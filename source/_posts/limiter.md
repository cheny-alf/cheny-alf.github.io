---
title: 常见的限流方案与实现
date: 2023-07-11 13:32:58
tags:
  - golang
  - redis
  - 限流
---
限流指的是系统在面对高并发、大流量的场景下，采用一些手段来限制新的流量对系统对访问，从而保证我们服务对稳定性。
常见对算法有 `计数器算法、漏斗算法以及令牌桶算法`，下面我们分别对这三种算法进行介绍与实现。
注：本次实现采用 golang + redis
### 限流方案
假设我们当前要完成一个关于限流的需求，要求某个接口一分钟内最多只能请求100次，并且采用redis实现。
#### 计数器算法
计数器算法相对来说是比较简单的一种限流算法，它又分为固定窗口和滑动窗口两种算法～ 
- 固定窗口
固定窗口算法指的是维护一个单位时间内的计数器counter，每个请求进来了就让counter + 1，如果在当前时间内，counter达到了阈值，就拒绝后面的请求，等待下一个时间窗口的时候再将counter重新置为0。对于我们的需求来说，就是维护一个单位时间为1分钟的时间窗口，如果在这一分钟内，counter的值超过了100，就拒绝后续的请求。等当前这个1分钟过去后再重置counter的值为0。
这个方法存在的问题：因为它的时间并不是动态的1分钟，而是固定好的时间线，所以如果在这一分钟的最后时间内突然有大量的请求，迅速打满了限制，随后就到了下一个一分钟
将计数器重置为0后又迅速的打满了限制。那当前这一分钟内的请求数量，达到了限制的两倍。即`临界值问题`。
具体实现如下
```go

```
- 滑动窗口
滑动窗口算法的是实现方式有很多种，比如说将时间窗口分成多个小单元，每个单元是很小的时间窗口，假设把60秒分成5个小单元，每个单元占12秒，然后再把每请求限制划分到小单元中，也就是每个单元20个请求，每过12秒将窗口向前移动一个单元。这样来实现动态的效果，但这个实现方式实际上还是固定窗口但实现方式，只是去将窗口变小了，
还是会存在上述的问题。
我们在这里提供另一种思路，就是采用redis的ZSet数据结构，我们将请求的时间戳作为score，将当前时间-60秒作为我们的时间窗口，窗口之前的请求全部删除掉，然后我们 zset获取到的数量就是我们单元时间内的计数值。这个窗口会随着时间而移动，就成为了我们的滑动窗口。
具体实现如下<br>
```go

```
#### 漏斗算法
我们可以从名字中得到关键字`漏斗`，它是用来比喻这个算法的实现思路。首先漏斗是具备一个固定的容量，同时它会漏水也可以进水，如果水满了，想再进水就要等漏水口把水流出。如果漏斗流水的速率 < 灌水的速率，那么漏斗最后将被灌满，漏斗满了就要等水流出去，腾出空间让其他的水进来。当然在我们的实现算法中，除了漏斗容量外还需要漏斗剩余空间，流水速率，上次流水时间。漏斗剩余空间代表当前情况下是否还可以继续处理请求，流水速率代表我们请求频率的限制，我们当前就是需要100/min，上次流水时间用于计算和当前时间的差值，然后用这个时间差计算这段时间漏斗流出了多少的水，腾出了多少的空间。<br>
具体实现如下<br>
```go

```
#### 令牌桶算法
令牌桶算法和漏斗限流实现起来大同小异，它以固定速度往一个桶内添加令牌，当桶内装满令牌后就停止。这个令牌即是我们的流量单位，按我们的需求来说就是拿到一个令牌就能发起一次请求。同时，我们的令牌桶结构还是能用 hash 类型存储。那么咱们废话少说，直接来看示例：<br>
具体实现如下<br>
```go

```
