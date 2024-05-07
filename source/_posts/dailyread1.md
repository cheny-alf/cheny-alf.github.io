---
title: 每日阅读--mutex的发展
date: 2023-08-16 13:36:31
tags:
- Golang
categories:
- 技术
---
解决并发访问的场景在我们的开发的过程中是非常常见的，比如多个goroutine同时访问同一个资源，如计数器、秒杀系统、往同一个buffer中写入数据等等，如果没有互斥控制，就会产生一些异常的状况，比如计数器计数不准，秒杀商品超卖，buffer中数据混乱等。

面对这些问题，在go语言中怎么解决呢？就是用`互斥锁`mutex！
### 前置知识
**临界区**:就是一个被共享的资源，或者说是一个整体的一组共享资源比如对数据库的访问、对某一个共享数据结构的操作、对一个 I/O 设备的使用、对一个连接池中的连接的调用等等。

**互斥锁**:就是保证在同一时刻只有一个线程可以访问我们的临界区。当临界区由一个线程持有的时候，其它线程如果想进入这个临界区，就会返回失败，或者是等待。直到持有的线程退出临界区，这些等待线程中的某一个才有机会接着持有这个临界区。

**并发原语**:原语，一般是指由若干条指令组成的程序段，用来实现某个特定功能，在执行过程中不可被中断。go官方的并发原语有goroutine、sync包下的Mutex、RWMutex、Once、WaitGroup、Cond、channel、Pool、Context、Timer、atomic等等

### 进入正题
假设我们当前有个场景是启动10个线程，每个线程去对count进行10万次+1操作。
我们在不加锁的情况下：
```go
import (
	"fmt"
	"sync"
)

func main() {
	var count = 0
	// 使用WaitGroup等待10个goroutine完成
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			// 对变量count执行10次加1
			for j := 0; j < 100000; j++ {
				count++
			}
		}()
	}
	// 等待10个goroutine完成
	wg.Wait()
	fmt.Println(count)
}
```
在这段代码中我们采用sync.WaitGroup来等待所有的协程都执行结束再输出最后的结果，但我们在每次运行上面但代码但时候，结果都会不尽如人意，没有得到我们想要都一百万，而接下来我们采用加锁的方式来试一下
```go
import (
    "fmt"
    "sync"
)
func main() {
    // 互斥锁保护计数器
    var mu sync.Mutex
    // 计数器的值
    var count = 0
    // 辅助变量，用来确认所有的goroutine都完成
    var wg sync.WaitGroup
    wg.Add(10)
    // 启动10个gourontine
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            // 累加10万次
            for j := 0; j < 100000; j++ {
                mu.Lock()
                count++
                mu.Unlock()
            }
        }()
	}
    wg.Wait()
    fmt.Println(count)
}
```
在这里我们可以看到，我们声明了一个互斥锁mutex，并且在每次操作count前对他进行Lock,而在操作结束后便进行UnLock。最后看运行结果，达到了我们期望的一百万。这是为什么呢？

其实是因为count++这个操作并不是原子操作，它包含很多步操作，比如先读取这个变量、再修改这个值、再将结果保存到count中。因为不是原子操作，所有在并发的情况下就会出现问题。比如10个协程同时读取到了count的值为8784，然后执行+1，变为8785，再将其写入到count变量中去，最终这个值是8785，但实际上它应该增加10才对，很多但计数操作都被"掩盖"掉了

所以我们通过加锁的方式，限制count只能被1个线程数操作。count++就是我们的临界区，我们只需在临界区前面加锁，在临界区后面释放掉锁就好了。