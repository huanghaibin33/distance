---
title: golang context 详解
translate_title: golang-context
date: 2019-07-25 17:33:45
categories:
- 编程语言
- go
tags:
- go
- context
---
### 为什么要用context
我们知道在golang 实现并发是通过goroutine 的方式，但在创建一个goroutine 时，并不会返回一个类似pid 的进程号，因此我们无法从外部去终止一个goroutine，只能通过WaitGroup 等待goroutine 执行完。这样会带来什么样的问题呢？我们先来看一个例子：
<!-- more -->
假设有一个用户请求，并发调用了下游的两个服务（RPC1，RPC2），然后再组合内容返回给用户。
正常情况下，两个RPC 调用都成功，如下图所示
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/context_normal.png" width="50%" height="50%">
但是如果RPC1 失败了，会发什么情况呢？最常用的做法则是等待RPC2 结束了，再返回失败给用户，这种情况RPC2 已经是在做无用功了，无论成功与否，对用户来说已经不重要了，而且还让用户白白多等待了一些时间。有一种比较高明的做法，那就是在RPC1 失败后立即返回，但是这种也存在一个缺陷，RPC2 还在运行，会造成资源的浪费。
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/context_fail.png">
<img src="https://public-1251890033.cos.ap-guangzhou.myqcloud.com/blog/go/context_opt.png" width="50%" height="50%">
最好的方式是有一种机制可以通知RPC2，让其提前退出（因为goroutine 不能被外部kill 掉），context 就很好的解决了这个问题，其实channel也能实现这个功能，但是需要额外的工作。说了这么多，context 到底是什么呢？
### context 是什么
go 语言的每一个请求都会开启一个单独的goroutine 来处理的，这个goroutine可能又会开启其他的goroutine，所以一个请求可能会经过多个goroutine，context 就是在这些goroutine 中传递一些信息，和控制信号，以便管理各个goroutine 的生命周期，中文可以称之为上下文。
### context 实战

``` go 
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// RPC ...
func RPC(ctx context.Context, method string, duration time.Duration) {
	beginTs := time.Now()
	// 统计实际耗时
	defer func() {
		fmt.Println(" cost ", time.Now().Sub(beginTs))
	}()

	select {
	case <-time.After(duration):
		// 正常结束
		fmt.Printf("%s succ", method)
	case <-ctx.Done():
		// 外部通知结束
		fmt.Printf("%s %s", method, ctx.Err())
	}
}

// Proccess ...
func Proccess(timeout time.Duration) {
	now := time.Now()
	defer func() {
		fmt.Println("Process cost ", time.Now().Sub(now))
	}()
	// 所有的context 都要从Background 开始
	ctx := context.Background()
	// 派生出新的context，定时取消context
	ctx, cancel := context.WithTimeout(ctx, timeout)
	// 在调用withTimeout、withCancel、WithDeadline后
	// 使用defer cancel 是一个不错的习惯
	// WithTimeout 定时到了也会执行cancel函数，不过cancel是幂等的
	defer cancel()

	wg := sync.WaitGroup{}
	wg.Add(2)

	// 开启两个goroutine 执行RPC： A和B
	go func() {
		RPC(ctx, "A", 100*time.Millisecond)
		wg.Done()
	}()
	go func() {
		RPC(ctx, "B", 300*time.Millisecond)
		wg.Done()
	}()
	wg.Wait()
}

func main() {
	Proccess(time.Millisecond * 200)
	// Proccess(time.Millisecond * 400)
}
```

运行的结果

``` go
A succ cost  100.198957ms
B context deadline exceeded cost  200.380496ms
Process cost  200.572052ms
Process exiting with code: 0
```

方法A 正常结束，方法B收到cancel 信号，主动结束goroutine返回，所以整个Process 的处理时间是200ms，如果将Process 的超时时间设置为400ms，则可以看到A、B 两个正常结束，而整体的耗时是300ms

```go 
A succ cost  100.191864ms
B succ cost  300.183041ms
Process cost  300.23551ms
Process exiting with code: 0
```

### context 源码分析
#### context 的接口定义

``` go 
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    // 返回cancel 的错误原因
    Err() error
    // 返回key 对应的value
    Value(key interface{}) interface{}
}
```
Context 的定义很简单，只有4个方法：
1. Deadline 返回是一个截止时间，如果没有设置deadline，则ok=false
1. Done 返回一个通道，当该通道可读时，则表示times out 或者调用cancel 关闭了通道
1. Err 返回一个error，表示cancel 的原因，超时或主动cancel
1. Value 获取Context 绑定的值，是一个键值对。通常用自定义类型作为key，而不是用string 类型，避免冲突

#### context 的实现
##### Background 和TODO

``` go 
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

Background通常用在主函数中，作为初始化节点。TODO根据官方的注释是在不清楚使用什么context时，可以使用这个，但是实际生产环境中，还没见过使用TODO的。
本质上，Background()和TODO()都返回一个emptyCtx，没有value，没有deadline，也不能被cancel掉。
##### WithValue

``` go 
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflect.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) String() string {
	return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

WithValue 比较简单，返回一个valueCtx，存了父context 和一个键值对，Value 方法根据key 返回对应的value，从当前节点开始向根节点回溯，直到找到对应的key，如果找不到对应的key，则返回nil。这里需要注意的一点是不应用基础类型string 作为key。
##### WithCancel

``` go 
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	// 初始化一个cancelCtx 实例
	c := newCancelCtx(parent)
	// ...
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}

type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```
WithCancel 是整个context 包的精髓，它返回了一个cancelCtx 实例和一个取消函数，这个函数就是用来关闭Done 方法返回的那个channel。propagateCancel 函数会找到第一个可cancel 的父context，然后把自己挂到父context 的children 里面取，当父context 调用cancel 时，自己也会跟着cancel。如果找不到，则表明自己是第一个可cancel 的context，这个时候会创建一个goroutine 出来，等待传入的父context 终止，则cancel 传入的child，或者等待传入的child 终止。如果不调用cancel 函数，那这个goroutine 就不会被终止，就会产生泄露。因此在使用withCancel方法后调用defer cancel() 是一个好的习惯，并且推荐这么做。

```go
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {
		return // parent is never canceled
	}
	// 向上找到第一个可cancel 的context
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent 已经被cancel 掉，则child 也需要跟着cancel
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			// 将child 挂到parent 的children 里面
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		// 创建一个goroutine 等待parent 或者child 终止
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

WithCancel 除了返回一个context，还返回一个取消函数CancelFunc，这个CancelFunc 就是调用了cancelCtx 的cancel 方法。主要功能就是关闭done 这个channel，并遍历所有children context，执行其cancel 方法（也就是关闭children 的done channel），最后将自己从parent context 移除

``` go 
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

##### WithTimeout 和WithDeadline
WithTimeout 和WithDeadline 是withCancel 的扩展，当到达设定时间时，会自动调用cancel。这两个函数也会返回CancelFunc，我们也可以自己提前cancel 掉

```go 
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(true, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
### 使用Context 的建议和技巧
1. 不要把Context 放在结构体中，要以参数的方式传递，parent Context 一般为Background
1. 应该要把Context 作为第一个参数传递给入口请求和出口请求链路上的每一个函数，
1. 变量名建议都统一，如ctx
1. Context 的Value 相关方法应该传递必须的数据，不要什么数据都使用这个传递
1. Context 是线程安全的，可以放心的在多个goroutine 中传递
1. 可以把一个Context 对象传递给任意个数的gorotuine，对它执行取消操作时，所有goroutine 都会接收到取消信号
1. 在使用withCancel 等可cancel 的函数后，应立即调用defer cancel()
