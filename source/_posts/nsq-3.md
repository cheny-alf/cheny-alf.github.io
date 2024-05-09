---
title: 深入NSQ--源码细节与实现(中)
date: 2023-11-2 17:41:14
tags:
- Golang
- nsq
categories:
- 技术
---
续上
接下来我们将介绍客户端的消费者 consumer 订阅、消费以及 ack 的流程
#### 1. 消费者
##### 1.1 类定义
```go
type Consumer struct {
    // ...
    // 互斥锁
    mtx sync.RWMutex

    // ...
    // 消费者标识 id
    id      int64
    // 消费者订阅的 topic
    topic   string
    // 消费者订阅的 channel
    channel string
    // ...
    // 用于接收消息的 channel
    incomingMessages chan *Message


    // ...
    pendingConnections map[string]*Conn
    connections        map[string]*Conn
  
    // 连接的 nsqd 地址
    nsqdTCPAddrs []string
    // ...
}
```
consumer 在被创建出来时，就需要明确指定其订阅的 topic 以及 channel.<br>
##### 1.2 添加 handler
用户可以通过调用 Consumer.AddHandler(...) 方法，针对每个 handler 会启动一个 handleLoop 协程，用于在消费到消息时执行预定义的回调函数.<br>
值得注意的是，如果用户注册了多个 handler ，最终每条消息只会随机被其中一个 handler 处理。
```go
func (r *Consumer) AddHandler(handler Handler) {
    r.AddConcurrentHandlers(handler, 1)
}
func (r *Consumer) AddConcurrentHandlers(handler Handler, concurrency int) {
    // ...
	// 设置并发数
    atomic.AddInt32(&r.runningHandlers, int32(concurrency))
    for i := 0; i < concurrency; i++ {
        go r.handlerLoop(handler)
    }
}
func (r *Consumer) handlerLoop(handler Handler) {
    // ...
    for {
        message, ok := <-r.incomingMessages
        // ...
        err := handler.HandleMessage(message)
        // ...
    }


    // ...
}
```
##### 1.3 连接服务器
消费者通过 consumer.ConnectToNSQD() 函数，实现与 nsqd 服务端建立连接，其步骤如下：
* 创建连接 NewConn()
* 建立连接 conn.Connect()
* 发布订阅指令 conn.WriteCommand(cmd)
```go
func (r *Consumer) ConnectToNSQD(addr string) error {
    // 创建链接
    conn := NewConn(addr, &r.config, &consumerConnDelegate{r})
    // ...
    r.mtx.Lock()
    _, pendingOk := r.pendingConnections[addr]
    _, ok := r.connections[addr]
    // ...
    r.pendingConnections[addr] = conn
    if idx := indexOf(addr, r.nsqdTCPAddrs); idx == -1 {
        r.nsqdTCPAddrs = append(r.nsqdTCPAddrs, addr)
    }
    r.mtx.Unlock()


    // ...
    resp, err := conn.Connect()
    // ...


    // ...
    cmd := Subscribe(r.topic, r.channel)
    err = conn.WriteCommand(cmd)
    // ...
    r.mtx.Lock()
    delete(r.pendingConnections, addr)
    r.connections[addr] = conn
    r.mtx.Unlock()
    // ...
    return nil
}
```
##### 1.4 消费消息
当指定 topic channel 下有新消息产生时，consumer 持有的 conn 会通过 readLoop goroutine 接收到对应的消息，并调用 Consumer.onConnMessage(...) 方法，将消息推送到 Consumer.incomingMessages 通道中.
```go
func (c *Conn) readLoop() {
    delegate := &connMessageDelegate{c}
    for {
        // ...


        frameType, data, err := ReadUnpackedResponse(c, c.config.MaxMsgSize)
        // ...


        switch frameType {
        // ...
        case FrameTypeMessage:
            msg, err := DecodeMessage(data)
            // ...           
            c.delegate.OnMessage(c, msg)
        // ...
    }


    // ...
}
func (d *consumerConnDelegate) OnMessage(c *Conn, m *Message)         { d.r.onConnMessage(c, m) }
func (r *Consumer) onConnMessage(c *Conn, msg *Message) {
    // ...
    r.incomingMessages <- msg
}
```
此前在注册 consumer handler 时启动的 handlerLoop goroutine 便会通过 Consumer.incomingMessages 通道获取到消息，并调用相应的 handler 执行回调处理逻辑.
```go
func (r *Consumer) handlerLoop(handler Handler) {
    // ...
    for {
        message, ok := <-r.incomingMessages
        // ...
        err := handler.HandleMessage(message)
        // ...
    }
    // ...
}
```
##### 1.6 消息 ack
为防止消息丢失，nsq 中同样支持了 consumer ack 机制<br>
consumer 在接收到消息并成功调用 handler 回调处理函数后，可以通过调用 Message.Finish() 方法，向服务端发送 ack 指令，确认消息已成功接收.<br>
倘若服务端超时未收到 ack 响应，则会默认消息已丢失，会重新推送一轮消息.<br>
consumer ack 流程的核心代码展示如下，在 Conn.onMessageFinish(...) 方法中，会通过 Conn.msgResponseChan 通道将数据推送到 Conn writeLoop goroutine 当中，由其负责将请求发往 nsqd 服务端.<br>
```go
func (m *Message) Finish() {
    .// ..
    m.Delegate.OnFinish(m)
}
func (d *connMessageDelegate) OnFinish(m *Message) { d.c.onMessageFinish(m) }
func (c *Conn) onMessageFinish(m *Message) {
    c.msgResponseChan <- &msgResponse{msg: m, cmd: Finish(m.ID), success: true}
}
```
Conn.writeLoop 是 Conn 负责向服务端发送指令的常驻 goroutine，其在接收到来自 Consumer.msgResponseChan 通道的数据后，会根据其成功状态，选择调用 OnMessageFinished 或者 OnMessageRequeued 将其封装成 ack 或者重试指令，最后调用 Conn.WriteCommand(...) 方法将指令发往服务端.
```go
func (c *Conn) writeLoop() {
    for {
        select {
        // ...
        // ...  
        case resp := <-c.msgResponseChan:
            // ...
            msgsInFlight := atomic.AddInt64(&c.messagesInFlight, -1)


            if resp.success {
                c.log(LogLevelDebug, "FIN %s", resp.msg.ID)
                c.delegate.OnMessageFinished(c, resp.msg)
                c.delegate.OnResume(c)
            } else {
                c.log(LogLevelDebug, "REQ %s", resp.msg.ID)
                c.delegate.OnMessageRequeued(c, resp.msg)
                if resp.backoff {
                    c.delegate.OnBackoff(c)
                } else {
                    c.delegate.OnContinue(c)
                }
            }


            err := c.WriteCommand(resp.cmd)
            // ...
        }
    }
    // ...
}
```