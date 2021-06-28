---
title: go语言上下文Context
tags:
  - 技术
  - Go语言
categories: 
  - 技术
  - Go语言
comments: true
cover: 'https://cdn.jsdelivr.net/gh/shimengfei/cdn/guidao/2.png'
abbrlink: 881e81b3
date: 2021-05-16 01:12:59
---
**本文章转载自**  {% btn 'https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/',draveness,far fa-hand-point-right,orange larger %}
---

# 1.1 上下文 Context
上下文 context.Context Go 语言中用来设置截止日期、同步信号，传递请求相关值的结构体。上下文与 Goroutine 有比较密切的关系，是 Go 语言中独特的设计，在其他编程语言中我们很少见到类似的概念。

context.Context 是 Go 语言在 1.7 版本中引入标准库的接口1，该接口定义了四个需要实现的方法，其中包括：

* Deadline — 返回 context.Context 被取消的时间，也就是完成工作的截止日期；
* Done — 返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消后关闭，多次调用 Done 方法会返回同一个 Channel；
* Err — 返回 context.Context 结束的原因，它只会在 Done 方法对应的 Channel 关闭时返回非空的值；
  * 如果 context.Context 被取消，会返回 Canceled 错误；
  * 如果 context.Context 超时，会返回 DeadlineExceeded 错误；

* Value — 从 context.Context 中获取键对应的值，对于同一个上下文来说，多次调用 Value 并传入相同的 Key 会返回相同的结果，该方法可以用来传递请求特定的数据；
```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```
context 包中提供的 ``context.Background``、``context.TODO``、``context.WithDeadline`` 和 ``context.WithValue`` 函数会返回实现该接口的私有结构体，我们会在后面详细介绍它们的工作原理。

## 1.1.1 设计原理
在 Goroutine 构成的树形结构中对信号进行同步以减少计算资源的浪费是 context.Context 的最大作用。Go 服务的每一个请求都是通过单独的 Goroutine 处理的2，HTTP/RPC 请求的处理器会启动新的 Goroutine 访问数据库和其他服务。

如下图所示，我们可能会创建多个 Goroutine 来处理一次请求，而 context.Context 的作用是在不同 Goroutine 之间同步请求特定数据、取消信号以及处理请求的截止日期。
![golang-context-usage.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/golang-context-usage.png)
图 1-1 Context 与 Goroutine 树

每一个 context.Context 都会从最顶层的 Goroutine 一层一层传递到最下层。context.Context 可以在上层 Goroutine 执行出现错误时，将信号及时同步给下层。

![golang-without-context.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/golang-without-context.png)
图 1-2 不使用 Context 同步信号

如上图所示，当最上层的 Goroutine 因为某些原因执行失败时，下层的 Goroutine 由于没有接收到这个信号所以会继续工作；但是当我们正确地使用 context.Context 时，就可以在下层及时停掉无用的工作以减少额外资源的消耗：
![golang-with-context.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/golang-with-context.png)

图 1-3 使用 Context 同步信号

我们可以通过一个代码片段了解 context.Context 是如何对信号进行同步的。在这段代码中，我们创建了一个过期时间为 1s 的上下文，并向上下文传入 handle 函数，该方法会使用 500ms 的时间处理传入的请求：
```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	go handle(ctx, 500*time.Millisecond)
	select {
	case <-ctx.Done():
		fmt.Println("main", ctx.Err())
	}
}

func handle(ctx context.Context, duration time.Duration) {
	select {
	case <-ctx.Done():
		fmt.Println("handle", ctx.Err())
	case <-time.After(duration):
		fmt.Println("process request with", duration)
	}
}
```
因为过期时间大于处理时间，所以我们有足够的时间处理该请求，运行上述代码会打印出下面的内容：
```go
$ go run context.go
process request with 500ms
main context deadline exceeded
```
handle 函数没有进入超时的 select 分支，但是 main 函数的 select 却会等待 context.Context 超时并打印出 main context deadline exceeded。

如果我们将处理请求时间增加至 1500ms，整个程序都会因为上下文的过期而被中止，：
```go
$ go run context.go
main context deadline exceeded
handle context deadline exceeded
```
相信这两个例子能够帮助各位读者理解 context.Context 的使用方法和设计原理 — 多个 Goroutine 同时订阅 ctx.Done() 管道中的消息，一旦接收到取消信号就立刻停止当前正在执行的工作。

## 1.1.2 默认上下文
context 包中最常用的方法还是 context.Background、context.TODO，这两个方法都会返回预先初始化好的私有变量 background 和 todo，它们会在同一个 Go 程序中被复用：
```go
func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```
这两个私有变量都是通过 new(emptyCtx) 语句初始化的，它们是指向私有结构体 context.emptyCtx 的指针，这是最简单、最常用的上下文类型：
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```
从上述代码中，我们不难发现 context.emptyCtx 通过空方法实现了 context.Context 接口中的所有方法，它没有任何功能。
![golang-context-hierarchy.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/golang-context-hierarchy.png)
图 1-4 Context 层级关系

从源代码来看，context.Background 和 context.TODO 也只是互为别名，没有太大的差别，只是在使用和语义上稍有不同：

context.Background 是上下文的默认值，所有其他的上下文都应该从它衍生出来；
context.TODO 应该仅在不确定应该使用哪种上下文时使用；
在多数情况下，如果当前函数没有上下文作为入参，我们都会使用 context.Background 作为起始的上下文向下传递。

## 1.1.3 取消信号
context.WithCancel 函数能够从 context.Context 中衍生出一个新的子上下文并返回用于取消该上下文的函数。一旦我们执行返回的取消函数，当前上下文以及它的子上下文都会被取消，所有的 Goroutine 都会同步收到这一取消信号。
![golang-parent-cancel-context.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/golang-parent-cancel-context.png)
图 1-5 Context 子树的取消

我们直接从 context.WithCancel 函数的实现来看它到底做了什么：
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```
``context.newCancelCtx`` 将传入的上下文包装成私有结构体 ``context.cancelCtx``；
``context.propagateCancel`` 会构建父子上下文之间的关联，当父上下文被取消时，子上下文也会被取消：
```go
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // 父上下文不会触发取消信号
	}
	select {
	case <-done:
		child.cancel(false, parent.Err()) // 父上下文已经被取消
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			child.cancel(false, p.err)
		} else {
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
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
上述函数总共与父上下文相关的三种不同的情况：

当 ``parent.Done() == nil``，也就是 ``parent`` 不会触发取消事件时，当前函数会直接返回；
当 ``child`` 的继承链包含可以取消的上下文时，会判断 ``parent`` 是否已经触发了取消信号；
如果已经被取消，``child`` 会立刻被取消；
如果没有被取消，``child`` 会被加入 ``parent`` 的 ``children`` 列表中，等待 ``parent`` 释放取消信号；
当父上下文是开发者自定义的类型、实现了 ``context.Context`` 接口并在 ``Done()`` 方法中返回了非空的管道时；
运行一个新的 Goroutine 同时监听 ``parent.Done()`` 和 ``child.Done()`` 两个 Channel；
在 ``parent.Done()`` 关闭时调用 ``child.cancel`` 取消子上下文；
``context.propagateCancel`` 的作用是在 ``parent`` 和 ``child`` 之间同步取消和结束的信号，保证在 ``parent`` 被取消时，``child`` 也会收到对应的信号，不会出现状态不一致的情况。

``context.cancelCtx`` 实现的几个接口方法也没有太多值得分析的地方，该结构体最重要的方法是 ``context.cancelCtx.cancel``，该方法会关闭上下文中的 Channel 并向所有的子上下文同步取消信号：
```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
除了 ``context.WithCancel`` 之外，``context`` 包中的另外两个函数 ``context.WithDeadline`` 和 ``context.WithTimeout`` 也都能创建可以被取消的计时器上下文 ``context.timerCtx``：
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // 已经过了截止日期
		return c, func() { c.cancel(false, Canceled) }
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

``context.WithDeadline`` 在创建 ``context.timerCtx`` 的过程中判断了父上下文的截止日期与当前日期，并通过 ``time.AfterFunc ``创建定时器，当时间超过了截止日期后会调用 ``context.timerCtx.cancel`` 同步取消信号。

``context.timerCtx`` 内部不仅通过嵌入 ``context.cancelCtx`` 结构体继承了相关的变量和方法，还通过持有的定时器 ``timer`` 和截止时间 ``deadline`` 实现了定时取消的功能：
```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
``context.timerCtx.cancel`` 方法不仅调用了 ``context.cancelCtx.cancel``，还会停止持有的定时器减少不必要的资源浪费。

## 1.1.4 传值方法
在最后我们需要了解如何使用上下文传值，context 包中的 context.WithValue 能从父上下文中创建一个子上下文，传值的子上下文使用 context.valueCtx 类型：
```go
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
Go
context.valueCtx 结构体会将除了 Value 之外的 Err、Deadline 等方法代理到父上下文中，它只会响应 context.valueCtx.Value 方法，该方法的实现也很简单：

type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```
如果 context.valueCtx 中存储的键值对与 context.valueCtx.Value 方法中传入的参数不匹配，就会从父上下文中查找该键对应的值直到某个父上下文中返回 nil 或者查找到对应的值。

## 1.1.5 小结
Go 语言中的 context.Context 的主要作用还是在多个 Goroutine 组成的树中同步取消信号以减少对资源的消耗和占用，虽然它也有传值的功能，但是这个功能我们还是很少用到。

在真正使用传值的功能时我们也应该非常谨慎，使用 context.Context 进行传递参数请求的所有参数一种非常差的设计，比较常见的使用场景是传递请求对应用户的认证令牌以及用于进行分布式追踪的请求 ID。

# 1.2 使用示例
## 1.2.1 在go程序中使用context进行取消操作

### 1.2.1.1 使用context取消的目的是什么？
简而言之，我们需要取消操作以防止我们的系统执行不必要的工作。例如HTTP服务器调用数据库并将查询的数据返回给客户端的常见情况：

![client-diagram.svg](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/client-diagram.svg)

1-6 客户端服务器模型图

* 如果一切正常，时序图将如下所示：
![timing-ideal.svg](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/timing-ideal.svg)

1-7 所有事件结束的时序图
* 但是，如果客户端在中间取消了请求，将会发生什么？
  * 例如，如果客户端在请求中关闭了浏览器，则可能会发生这种情况。如果不取消，则应用程序服务器和数据库将继续执行其工作，即使这样做会浪费工作的结果：
![timing-without-cancel.svg](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/timing-without-cancel.svg)
1-8 取消HTTP请求后所有进程继续执行的时序图
  * 理想情况下，如果我们知道该进程（在此示例中为HTTP请求）已停止，则我们希望该进程的所有下游组件都停止运行：
![timing-with-cancel.svg](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/timing-with-cancel.svg)
1-9 取消HTTP请求后所有进程都将取消的时序图

### 1.2.1.2 使用context取消
现在我们知道了为什么需要取消，让我们进入如何在Go中实现它。由于“取消”事件与交易或正在执行的操作高度相关，因此很自然地将其与捆绑在一起context。

cancel有两个方面：
* 监听取消事件
* 发出取消事件

#### 监听取消事件
该Context类型提供一种Done()方法，struct{}每次上下文接收到取消事件时，该方法都会返回一个接收空类型的通道。聆听取消事件就像等待一样容易<- ctx.Done()。

例如，让我们考虑一个需要花费两秒钟来处理事件的HTTP服务器。如果在此之前请求被取消，我们想立即返回：
```go
func main() {
	// Create an HTTP server that listens on port 8000
	http.ListenAndServe(":8000", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		// This prints to STDOUT to show that processing has started
		fmt.Fprint(os.Stdout, "processing request\n")
		// We use `select` to execute a peice of code depending on which
		// channel receives a message first
		select {
		case <-time.After(2 * time.Second):
			// If we receive a message after 2 seconds
			// that means the request has been processed
			// We then write this as the response
			w.Write([]byte("request processed"))
		case <-ctx.Done():
			// If the request gets cancelled, log it
			// to STDERR
			fmt.Fprint(os.Stderr, "request cancelled\n")
		}
	}))
}

```

#### 发出取消事件
如果有可以取消的操作，则必须通过上下文发出取消事件。可以使用上下文包中的WithCancel函数（该函数返回一个上下文对象）和一个函数来完成此操作。此函数不带任何参数，不返回任何内容，当您要取消上下文时会调用此函数。
考虑2个从属操作的情况。在这里，“依赖”意味着如果一个失败，那么另一个就没有意义。在这种情况下，如果我们很早就知道其中一个操作失败，那么我们想取消所有相关的操作。

```go

func operation1(ctx context.Context) error {
	// Let's assume that this operation failed for some reason
	// We use time.Sleep to simulate a resource intensive operation
	time.Sleep(100 * time.Millisecond)
	return errors.New("failed")
}

func operation2(ctx context.Context) {
	// We use a similar pattern to the HTTP server
	// that we saw in the earlier example
	select {
	case <-time.After(500 * time.Millisecond):
		fmt.Println("done")
	case <-ctx.Done():
		fmt.Println("halted operation2")
	}
}

func main() {
	// Create a new context
	ctx := context.Background()
	// Create a new context, with its cancellation function
	// from the original context
	ctx, cancel := context.WithCancel(ctx)

	// Run two operations: one in a different go routine
	go func() {
		err := operation1(ctx)
		// If this operation returns an error
		// cancel all operations using this context
		if err != nil {
			cancel()
		}
	}()

	// Run operation2 with the same context we use for operation1
	operation2(ctx)
}

```

#### 基于时间的取消事件
例如，对外部服务进行HTTP API调用。如果服务花费的时间太长，最好尽早失败并取消请求：
```go
func main() {
	// Create a new context
	// With a deadline of 100 milliseconds
	ctx := context.Background()
	ctx, _ = context.WithTimeout(ctx, 100*time.Millisecond)

	// Make a request, that will call the google homepage
	req, _ := http.NewRequest(http.MethodGet, "http://google.com", nil)
	// Associate the cancellable context we just created to the request
	req = req.WithContext(ctx)

	// Create a new HTTP client and execute the request
	client := &http.Client{}
	res, err := client.Do(req)
	// If the request failed, log to STDOUT
	if err != nil {
		fmt.Println("Request failed:", err)
		return
	}
	// Print the statuscode if the request succeeds
	fmt.Println("Response received, status code:", res.StatusCode)
}

```
### 1.2.1.3 小结
尽管Go中的上下文取消功能是一种多功能工具，但是在继续操作之前，需要牢记一些注意事项。其中最重要的是，上下文只能被取消一次。如果希望在同一操作中传播多个错误，那么使用上下文取消可能不是最佳选择。使用取消的最惯用的方式是当实际上要取消某项时，而不仅仅是通知下游进程发生了错误。
还需要记住的另一件事是，应将相同的上下文实例传递给可能要取消的所有函数和例程。用``WithTimeout``或``WithCancel``包装已经可以取消的上下文会导致取消上下文的多种可能性，应该避免。
