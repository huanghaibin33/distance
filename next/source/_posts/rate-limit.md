---
title: 常见的限流算法
translate_title: common-current-limiting-algorithm
categories:
  - 限流
tags:
  - 漏桶算法
  - 令牌桶算法
date: 2019-08-07 15:52:06
---
### 问题引入
试想下天猫这个网站，在双十一和双十二等大型促销活动，访问量暴涨，如果不对流量进行控制牵引，任其冲击系统，那么有可能会击穿数据库，导致系统奔溃而无法访问网站。所以为了保护系统的正常运行，需要对外部流量就行限制，对超出系统能力范围的请求提供有损服务或者简单粗暴的将拒绝服务。
### 限流算法
目前流行的限流算法有以下几种：
1. 计数器
1. 滑动窗口
1. 漏桶
1. 令牌桶

<!-- more -->
### 计数器
计数器限流算法是常用的、也是比较简单的。在一个时间段内对请求数量进行计数，如果请求超过某个阈值，就会被拒绝掉或者是排队等待服务，当到了下个时间段，计数器会清零重新计数。
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/rate_limit/counter.png" widtch="50%" height="50%">
这种算法最大的优点就是简单，缺点就是不能均衡流量。考虑两种情况：
1. 流量特别大，在第1ms就到达了阈值，那么后面999ms 的流量将会被拒绝
1. 假设第1s 前面999ms没有流量（或少量），在最后1ms 涌入大量流量，并且在第2s 的第1ms 也有大量流量，那在这2ms 时间段内的流量有可能是系统阈值的2倍。
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/rate_limit/counter_abnormal.png">

### 滑动窗口
滑动窗口是将计数器的时间段（窗口）进行分片，比如将1s 划分为10个分片，每个分片都有自己独立的计数器，所有分片的计数超过阈值就会被限流。所以滑动窗口本质上也是一种计数器，只不过它的粒度更细。滑动窗口能有效的解决毛刺问题，而且分片越多，平滑效果越好。
### 漏桶
漏桶算法思路很简单，水（请求）先进入到漏桶里，漏桶以一定的速度出水，当水流入速度过大会直接溢出，可以看出漏桶算法能强行限制数据的传输速率. 
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/rate_limit/leaky_bucket.png" width="50%" height="50%">
注：图片来源网络
go 参考代码

``` go 
package ratelimiter

import (
	"sync"
	"time"
)

// LeakyBucket ...
type LeakyBucket struct {
	sync.Mutex
	rate   float64          // rate
	cap    uint64           // capacity
	water  chan interface{} // storage
	ticker *time.Ticker
}

// NewLeakBucket ...
func NewLeakBucket(rate float64, cap uint64) *LeakyBucket {
	lb := LeakyBucket{
		rate:   rate,
		cap:    cap,
		water:  make(chan interface{}, cap),
		ticker: time.NewTicker(time.Duration(1000/rate) * time.Millisecond),
	}
	return &lb
}

// Inflow ...
func (lb *LeakyBucket) Inflow(req interface{}) bool {
	lb.Lock()
	defer lb.Unlock()
	if uint64(len(lb.water)) < lb.cap {
		lb.water <- req
		return true
	}
	return false
}

// Drip ...
func (lb *LeakyBucket) Drip() interface{} {
	select {
	case <-lb.ticker.C:
		x := <-lb.water
		return x
	}
}
```

### 令牌桶
令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，如果有请求到来，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。从原理上看，令牌桶算法和漏桶算法是相反的，一个“进水”，一个是“漏水”。
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/rate_limit/token_bucket.png" width="50%" height="50%">
注：图片来源网络
go 参考代码
``` go 
package ratelimiter

import (
	"sync"
	"time"
)

// TokenBucket ...
type TokenBucket struct {
	sync.Mutex
	lastTs   time.Time
	capacity uint64
	token    float64
	rate     float64
}

// NewTokenBucket ...
func NewTokenBucket(rate float64, capacity uint64) *TokenBucket {
	tb := TokenBucket{
		lastTs:   time.Now(),
		capacity: capacity,
		token:    float64(capacity),
		rate:     rate,
	}
	return &tb
}

// Check ...
func (tb *TokenBucket) Check(n uint64) bool {
	tb.Lock()
	defer tb.Unlock()
	nowTs := time.Now()
	tb.token += tb.rate * nowTs.Sub(tb.lastTs).Seconds()
	tb.lastTs = nowTs
	if tb.token > float64(tb.capacity) {
		tb.token = float64(tb.capacity)
	}

	if tb.token > float64(n) {
		tb.token = tb.token - float64(n)
		return true
	}
	return false
}
```

### 总结
限流技术是保护系统正常运行的一种重要手段，在大流量场景中不可或缺。常见的限流算法有4中，其中计数器是比较简单的，但是会出现毛刺和超过系统阈值的风险。滑动窗口能有效的平滑，但平滑的效果取决于分片的大小。漏桶算法能很好的控制流量的访问速度，但它对于存在突发特性的流量来说缺乏效率，而令牌桶既能很好的控制流量的访问速度，也能应对突发特性的流量。
