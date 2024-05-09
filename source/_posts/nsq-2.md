---
title: 深入NSQ--源码细节与实现(上)
date: 2023-11-1 17:41:14
tags:
- Golang
- nsq
categories:
- 技术
---
接下来，我们将深入到 [nsq 客户端的源码](https://github.com/nsqio/go-nsq)中，梳理生产者与消费者的运行流程<br>
#### 1. 连接
在客户端与服务端交互时，无论是生产者还是消费者，都是通过客户端定义的 Conn 类来进行两端的连接建立。
##### 1.1 类定义
```go
type Conn struct {
	// 记录有多少消息还没确认
	messagesInFlight int64
	// 互斥锁，保证临界资源一致性
	mtx sync.Mutex 
	// 真正的 tcp 连接
	conn    *net.TCPConn
	// 服务端地址
	addr    string 
	// 读
	r io.Reader
	// 写
	w io.Writer 
	// 用与接收用户指令的channel
	cmdChan         chan *Command
	// 通过此 channel 接收来自客户端的响应，发往服务端
	msgResponseChan chan *msgResponse
	// 控制 write goroutine 退出的 channel
	exitChan        chan int
	// 并发等待组，保证所有 goroutine 及时回收，conn 才能关闭
	wg        sync.WaitGroup
}
```
##### 1.2 创建连接
由客户端像服务端发起一笔连接，实际是通过 conn.Connect()，其核心流程如下
* 通过 net 包向服务端发起tcp连接
* 将 conn 下的 writer 和 reader 置为此tcp连接
* 异步启动 readLoop 和 writeLoop 协程，持续监听客户端与服务端的交互
```go

func (c *Conn) Connect() (*IdentifyResponse, error) {
	dialer := &net.Dialer{
		LocalAddr: c.config.LocalAddr,
		Timeout:   c.config.DialTimeout,
	}

	conn, err := dialer.Dial("tcp", c.addr)
	if err != nil {
		return nil, err
	}
	c.conn = conn.(*net.TCPConn)
	c.r = conn
	c.w = conn 
	// 魔数检查
	_, err = c.Write(MagicV2)
	if err != nil {
		c.Close()
		return nil, fmt.Errorf("[%s] failed to write magic - %s", c.addr, err)
	}
	// ...
    
	c.wg.Add(2)
	atomic.StoreInt32(&c.readLoopRunning, 1)
	// 接收来自服务端的相应
	go c.readLoop()
	// 向服务端发送请求
	go c.writeLoop()
	return resp, nil
}
```
关于 readLoop 与 writeLoop 的设计,其核心优势在于：<br>
*  **解耦客户端发送请求与服务端响应**的串行流程，能够更加自由的双工通信。
* 充分利用 goroutine 和 channel 的优势，通过 **for + select** 的多路复用，保证两个 loop goroutine 能够在监听到退出指令时优雅退出。比如 context 的 cancel、timeout 事件，比如 exitChan 的关闭事件等。
<p>
readLoop 与 writeLoop 具体内容会在后面写到

#### 2 生产者
下面我们会在客户端的视角下梳理下生产者 producer 向服务端生产消息的流程
##### 2.1 类定义
```go
type Producer struct {
	// 生产者标识 id
	id     int64
	// 连接的 nsqd 服务器地址
	addr   string
	// 内置的客户端连接，producerConn是一个抽象类，具体的实现为上面的Conn
	conn   producerConn
	// ... 
	
	// 接收服务端响应的 channel 
	responseChan chan []byte
	// 接收错误的 channel
	errorChan    chan []byte
	// 接收生产者关闭信号的 channel
	closeChan    chan int 
	// 接收 publish 指令的 channel
	transactionChan chan *ProducerTransaction
	transactions    []*ProducerTransaction
	// 生产者的状态
	state           int32
	// 当前有多少 publish 指令并发执行
	concurrentProducers int32
	stopFlag            int32
	// 接收退出信号的 channel
	exitChan            chan int 
	// 并发等待组，保证 producer 关闭前及时回收所有 goroutine
	wg                  sync.WaitGroup
	// 互斥锁
	guard               sync.Mutex
}
```
##### 2.2 publish 指令
正如我们提供的示例中所示，生产者 producer 生产消息的入口函数为 `producer.Publish(testTopic, []byte(msg))`，其底层会组装出一个PUB指令，并调用 sendCommand() 函数发送指令。<br>
```go
func (w *Producer) Publish(topic string, body []byte) error {
	return w.sendCommand(Publish(topic, body))
}
```
生成 PUB 指令
```go
func Publish(topic string, body []byte) *Command {
	var params = [][]byte{[]byte(topic)}
	return &Command{[]byte("PUB"), params, body}
}
```
##### 2.3 发送指令
* 如果当前 producer 的 state 字段不是已连接状态(即首次发送)，则会调用 Connect() 函数与服务端建立连接
* 将指令写入 transactionChan 通道中
* router 函数中，接收到数据后，调用 conn.WriteCommand() 将指令发往服务端
```go
func (w *Producer) sendCommand(cmd *Command) error {
	doneChan := make(chan *ProducerTransaction)
	err := w.sendCommandAsync(cmd, doneChan, nil)
	// ...
	t := <-doneChan
	return t.Error
}
```

```go
func (w *Producer) sendCommandAsync(cmd *Command, doneChan chan *ProducerTransaction,
	args []interface{}) error {
    // 检查当前 Producer 的状态
	if atomic.LoadInt32(&w.state) != StateConnected {
		err := w.connect()
	}

	t := &ProducerTransaction{
        cmd:      cmd,
        doneChan: doneChan,
        Args:     args,
	}
    select {
    case w.transactionChan <- t:
    case <-w.exitChan:
        return ErrStopped
    }
	return nil
}
```
在 producer.connect() 函数中，与服务端建立连接的过程流程如下：
* 创建 conn 实例
* 调用 conn.Connect() 向目标地址发起连接（connect() 流程上文已经介绍）
* 异步启动 router() 函数，负责持续接收指令，通过 conn 与服务端交互
```go
func (w *Producer) connect() error {
    w.guard.Lock()
    defer w.guard.Unlock()
	
    // ...
    w.conn = NewConn(w.addr, &w.config, &producerConnDelegate{w})
    // ...
    // producer 创建和 nsqd 之间的连接
    _, err := w.conn.Connect()
    // ...
    atomic.StoreInt32(&w.state, StateConnected)
    w.closeChan = make(chan int)
    w.wg.Add(1)
    go w.router()
	
    return nil
}
```
router goroutine 的运行框架如下，主要通过 for + select 的形式，持续接收来自 Producer.transactionChan 通道的指令，将其发往服务端
```go
func (w *Producer) router() {
    for {
        select {
        case t := <-w.transactionChan:
            w.transactions = append(w.transactions, t)
            err := w.conn.WriteCommand(t.cmd)
            // ...
        // ...
        case <-w.closeChan:
            goto exit
        case <-w.exitChan:
            goto exit
        }
    }


// ...
}
```
##### 2.4 接收响应
producer 在发送完 PUB 指令后，又要如何接收到来自服务端的响应呢.流程如下
* 在 conn.readLoop() 方法中，读取到来自服务端的响应。通过调用 Producer.onConnResponse() 方法，将数据发送到 Producer.responseChan 当中
```go
func (c *Conn) readLoop() {
    delegate := &connMessageDelegate{c}
    for {
        // ...


        frameType, data, err := ReadUnpackedResponse(c, c.config.MaxMsgSize)
        // ...


        switch frameType {
        case FrameTypeResponse:
            c.delegate.OnResponse(c, data)
        // ...
        }
    }
    // ...
}
func (d *producerConnDelegate) OnResponse(c *Conn, data []byte)       { d.w.onConnResponse(c, data) }
func (w *Producer) onConnResponse(c *Conn, data []byte) { w.responseChan <- data }
```
* Producer 的 router goroutine 通过 Producer.responseChan 接收到响应数据，通过调用 Producer.popTransaction(...) 方法，将响应推送到 doneChan 当中
```go
func (w *Producer) router() {
    for {
        select {
        // ...
        case data := <-w.responseChan:
            w.popTransaction(FrameTypeResponse, data)
        case data := <-w.errorChan:
            w.popTransaction(FrameTypeError, data)
        // ...
    }


    // ...
}
func (w *Producer) popTransaction(frameType int32, data []byte) {
    // ...
    t := w.transactions[0]
    // ...
    t.finish()
}
func (t *ProducerTransaction) finish() {
    if t.doneChan != nil {
        t.doneChan <- t
    }
}
```
* 客户端通过在 Producer.sendCommand 方法中阻塞等待来自 doneChan 的数据，接收到后将其中的错误返回给上层.
```go
func (w *Producer) sendCommand(cmd *Command) error {
    doneChan := make(chan *ProducerTransaction)
    err := w.sendCommandAsync(cmd, doneChan, nil)
    // ...
    t := <-doneChan
    return t.Error
}
```
至此，生产者的全部流程就梳理完了。