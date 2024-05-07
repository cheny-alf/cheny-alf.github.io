---
title: nsq-1
date: 2023-10-30 17:41:14
tags:
- Golang
- nsq
categories:
- 技术
---
#### 1. 初识
`NSQ is a realtime distributed messaging platform.` nsq 是一款基于 GO 实现的分布式消息队列，
它可用于大规模系统中的实时消息服务，并且每天能够处理数亿级别的消息。<br>
#### 2. NSQ特性
* **分布式架构**: NSQ采用无单点故障的分布式和去中心化拓扑结构，确保了系统的高容错性和高可用性，同时提供了可靠的消息传递保证。
* **水平扩展性**: NSQ能够水平扩展，无需依赖任何中央代理，其内置的发现机制简化了集群中节点的添加，支持发布-订阅和负载均衡的消息传递方式，性能同样出色。
* **运维友好**: NSQ易于配置和部署，带有管理界面，其二进制文件无需运行时依赖，提供针对Linux、Darwin、FreeBSD和Windows的预编译版本，以及官方的Docker镜像。
* **集成性**: NSQ提供了官方的Go和Python库，以及社区支持的多种主流语言库（详见客户端库），同时，如果用户有兴趣自行构建，还可以访问协议规范文档。
#### 3. NSQ使用
1.  服务安装与启动<br>
[官方文档](https://nsq.io/deployment/installing.html)中有很多的安装方式，我们选择最后一个，源码启动
```go
$ git clone https://github.com/nsqio/nsq
$ cd nsq
$ make // 编译
$ cd build
$ ./nsqlookupd & // 启动 nsqlookupd
$ ./nsqd --lookupd-tcp-address=127.0.0.1:4160 & // 启动nsqd
$ ./nsqadmin --lookupd-http-address=127.0.0.1:4161 & // 启动nsqadmin
```
在nsqadmin启动后即可在浏览器中输入 http://localhost:4171 访问管理后台，查看具体的 topic、channel 信息。
2.  编写客户端
```go
package main

import (
	"fmt"
	"sync"
	"testing"
	"time"

	"github.com/nsqio/go-nsq"
)

const (
	// 用于测试的消息主题
	testTopic = "my_test_topic"
	// nsqd 服务端地址
	nsqdAddr = "localhost:4150"
	// 订阅 channelGroupA 的消费者 A1
	consumerA1 = "consumer_a1"
	// 订阅 channelGroupA 的消费者 A2
	consumerA2 = "consumer_a2"
	// 单独订阅 channelGroupB 的消费者 B
	consumerB = "consumer_b"
	// testTopic 下的 channelA
	channelGroupA = "channel_a"
	// testTopic 下的 channelB
	channelGroupB = "channel_b"
)

// 建立 consumer 与 channel 之间的映射关系
// consumerA1、consumerA2 -> channelA
// consumerB -> channelB
var consumerToChannelGroup = map[string]string{
	consumerA1: channelGroupA,
	consumerA2: channelGroupA,
	consumerB:  channelGroupB,
}

func Test_nsq(t *testing.T) {
	// 消费者消费到消息后的执行逻辑
	msgCallback := func(consumerName string, msg []byte) error {
		t.Logf("i am %s, receive msg: %s", consumerName, msg)
		return nil
	}

	// 运行 producer
	if err := runProducer(); err != nil {
		t.Error(err)
		return
	}

	// 并发运行三个 consumer
	var wg sync.WaitGroup
	for consumer := range consumerToChannelGroup {
		// shadow
		wg.Add(1)
		go func(consumer string) {
			defer wg.Done()
			if err := runConsumer(consumer, msgCallback); err != nil {
				t.Error(err)
			}
		}(consumer)

	}
	wg.Wait()
}

// 运行生产者
func runProducer() error {
	// 通过 addr 直连 nsqd 服务端
	producer, err := nsq.NewProducer(nsqdAddr, nsq.NewConfig())
	if err != nil {
		return err
	}
	defer producer.Stop()

	// 通过 producer.Publish 方法，往 testTopic 中发送三条消息
	for i := 0; i < 3; i++ {
		msg := fmt.Sprintf("hello xiaoxu %d", i)
		if err := producer.Publish(testTopic, []byte(msg)); err != nil {
			return err
		}
	}
	return nil
}

// 用于处理消息的 processor，需要实现 go-nsq 中定义的 msgProcessor interface，核心是实现消息回调处理方法： func HandleMessage(msg *nsq.Message) error
type msgProcessor struct {
	// 消费者名称
	consumerName string
	// 消息回调处理函数
	callback func(consumerName string, msg []byte) error
}

func newMsgProcessor(consumerName string, callback func(consumerName string, msg []byte) error) *msgProcessor {
	return &msgProcessor{
		consumerName: consumerName,
		callback:     callback,
	}
}

// 消息回调处理
func (m *msgProcessor) HandleMessage(msg *nsq.Message) error {
	// 执行用户定义的业务处理逻辑
	if err := m.callback(m.consumerName, msg.Body); err != nil {
		return err
	}
	// 倘若业务处理成功，则调用 Finish 方法，发送消息的 ack
	msg.Finish()
	return nil
}

// 运行消费者
func runConsumer(consumerName string, callback func(consumerName string, msg []byte) error) error {
	// 根据消费者名获取到对应的 channel
	channel, ok := consumerToChannelGroup[consumerName]
	if !ok {
		return fmt.Errorf("bad name: %s", consumerName)
	}

	// 指定 topic 和 channel，创建 consumer 实例
	consumer, err := nsq.NewConsumer(testTopic, channel, nsq.NewConfig())
	if err != nil {
		return err
	}
	defer consumer.Stop()

	// 添加消息回调处理函数
	consumer.AddHandler(newMsgProcessor(consumerName, callback))

	// consumer 连接到 nsqd 服务端，开启消费流程
	if err = consumer.ConnectToNSQD(nsqdAddr); err != nil {
		return err
	}

	<-time.After(5 * time.Second)
	return nil
}
```
3.  测试
```go
nsq_test.go:41: i am consumer_a2, receive msg: hello xiaoxu 1
nsq_test.go:41: i am consumer_b, receive msg: hello xiaoxu 0
nsq_test.go:41: i am consumer_a1, receive msg: hello xiaoxu 0
nsq_test.go:41: i am consumer_b, receive msg: hello xiaoxu 1
nsq_test.go:41: i am consumer_a2, receive msg: hello xiaoxu 2
nsq_test.go:41: i am consumer_b, receive msg: hello xiaoxu 2

```
可以看到，共享了 channelA 的两个 consumerA1、consumerA2 分摊了 topic 下的消息数据；而独享 channelB 的 consumer 则消费到了 topic 下的全量消息.
