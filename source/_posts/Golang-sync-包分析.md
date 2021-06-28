---
title: Golang sync 包分析
tags:
  - 技术
  - Go语言
categories: 
  - 技术
  - Go语言
comments: true
abbrlink: 8d0aa32e
date: 2021-03-24 22:54:46
cover: https://cdn.jsdelivr.net/gh/shimengfei/cdn/guidao/2.png
---
## 第一章 Golang sync 包的相关使用方法

### 一、锁的基本概念及使用

整个包都是实现Locker接口

~~~go
type Locker interface{
  Lock()
  UnLock()
}
~~~

该接口只有两个方法，`Lock()` 和`UnLock()` 。注意：`该包下的对象时候过后不可进行复制` 

#### 为什么需要锁？

首先，我们都知道在**并发** 的情况下，多线程或者协程同时修改一个变量的时候时，可能会出现如下情况:

~~~go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var a = 0

    // 启动 100 个协程，需要足够大
    // var lock sync.Mutex
    for i := 0; i < 100; i++ {
        go func(idx int) {
            // lock.Lock()
            // defer lock.Unlock()
            a += 1
            fmt.Printf("goroutine %d, a=%d\n", idx, a)
        }(i)
    }

    // 等待 1s 结束主程序
    // 确保所有协程执行完
    time.Sleep(time.Second)
}
~~~

根据如上的代码运行，是否会出现打印出来的a值有相同的情况？答案是肯定的，会出现a值相同的情况。

其实原因很简单，a是全局变量。多协程同时进行操作时，会出现类似于`脏读`的场景。所以`锁`的概念其实就是，我正在操作a（加锁），其他人谁都不要和我抢，我处理完了（解锁），其他人在进行相关处理。这样其实就是保证了，同时操作a的协程只有一个，这样就实现了同步。

{% note success simple %}
可以将上面的注释取消，体会一下锁的使用
{% endnote %}

#### 什么是互斥锁？

什么是互斥锁？

~~~go
共享资源的使用是互斥的，即一个线程获得资源的使用权后就会将该资源加锁，使用完后会将其解锁，如果在使用过程中有其他线程想要获取该资源的锁，那么它就会被阻塞陷入睡眠状态，直到该资源被解锁才会被唤醒，如果被阻塞的资源不止一个，那么它们都会被唤醒，但是获得资源使用权的是第一个被唤醒的线程，其它线程又陷入沉睡.
~~~

互斥锁是锁的一种具体实现，

~~~go
func (m *Mutex)Lock()
func (m *Mutex)UnLock()
~~~

注意：**在首次使用后不要复制该锁，对于一个未锁定的互斥锁进行解锁会抛出运行时错误** 。一个互斥锁只能同时被一个goroutine锁定，其他goroutine

会被阻塞，直到该锁被释放。如下，

~~~go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    ch := make(chan struct{}, 2)

    var l sync.Mutex
    go func() {
        l.Lock()
        defer l.Unlock()
        fmt.Println("goroutine1: 我会锁定大概 2s")
        time.Sleep(time.Second * 2)
        fmt.Println("goroutine1: 我解锁了，你们去抢吧")
        ch <- struct{}{}
    }()

    go func() {
        fmt.Println("groutine2: 等待解锁")
        l.Lock()
        defer l.Unlock()
        fmt.Println("goroutine2: 哈哈，我锁定了")
        ch <- struct{}{}
    }()

    // 等待 goroutine 执行结束
    for i := 0; i < 2; i++ {
        <-ch
    }
}
~~~

{% note success simple %}
注意，平时所说的锁定，其实就是去锁定互斥锁，而不是说去锁定一段代码。也就是说，当代码执行到有锁的地方时，它获取不到互斥锁的锁定，会阻塞在那里，从而达到控制同步的目的。
{% endnote %}

#### 什么是读写锁 RWMutex？

在互斥锁中, 只有两个状态: 加锁和未加锁, 而在一些情况下对于`读`可以并发的进行而不用加锁, 对于读则需要加锁, 比如golang中map的操作。

为了让`读`操作更快的进行(不必加锁), 就诞生了`读写锁`的概念, 它有三个状态: 读模式下加锁状态, 写模式加锁状态和未加锁状态。

规则如下

- 如果有其它协程读数据, 则允许其它协程执行读操作, 但不允许写操作
- 如果有其它协程写数据, 则其它协程都不允许读和写操作

由于这个特性, 读写锁能在读频率更高的情况下有更好的并发性能。

~~~go
func (rw *RWMutex) Lock()
func (rw *RWMutex) Unlock()

func (rw *RWMutex) RLock()
func (rw *RWMutex) RUnlock()
~~~

由于这里需要区分读写锁定，：

- 读锁定（RLock），对读操作进行锁定
- 读解锁（RUnlock），对读锁定进行解锁 
- 写锁定（Lock），对写操作进行锁定
- 写解锁（Unlock），对写锁定进行解锁

注意：**在首次使用之后，不要复制该读写锁。不要混用锁定和解锁，如：Lock 和 RUnlock、RLock 和 Unlock。因为对未读锁定的读写锁进行读解锁或对未写锁定的读写锁进行写解锁将会引起运行时错误。** 

~~~go
package main

import (
    "fmt"
    "math/rand"
    "sync"
)

var count int
var rw sync.RWMutex

func main() {
    ch := make(chan struct{}, 10)
    for i := 0; i < 5; i++ {
        go read(i, ch)
    }
    for i := 0; i < 5; i++ {
        go write(i, ch)
    }

    for i := 0; i < 10; i++ {
        <-ch
    }
}

func read(n int, ch chan struct{}) {
    rw.RLock()
    fmt.Printf("goroutine %d 进入读操作...\n", n)
    v := count
    fmt.Printf("goroutine %d 读取结束，值为：%d\n", n, v)
    rw.RUnlock()
    ch <- struct{}{}
}

func write(n int, ch chan struct{}) {
    rw.Lock()
    fmt.Printf("goroutine %d 进入写操作...\n", n)
    v := rand.Intn(1000)
    count = v
    fmt.Printf("goroutine %d 写入结束，新值为：%d\n", n, v)
    rw.Unlock()
    ch <- struct{}{}
}
~~~

#### WaitGroup 使用

WaitGroup用于等待一组goroutine结束。

~~~go
func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()
~~~

Add用来添加goroutine的数量，Done()代表一个协程处理完成，等待数量减一。Wait 用来等待结束，也即未完成goroutine数量为0。

~~~go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        // 计数加 1
        wg.Add(1)
        go func(i int) {
            // 计数减 1
            defer wg.Done()
            time.Sleep(time.Second * time.Duration(i))
            fmt.Printf("goroutine%d 结束\n", i)
        }(i)
    }

    // 等待执行结束
    wg.Wait()
    fmt.Println("所有 goroutine 执行结束")
}
~~~

注意：`wg.Add()`一定要在goroutine初始化前使用。

#### Cond 条件变量

Cond 实现一个条件变量，即等待或宣布事件发生的 goroutines 的会合点，它会保存一个通知列表。基本思想是当某中状态达成，goroutine 将会等待（Wait）在那里，当某个时刻状态改变时通过通知的方式（Broadcast，Signal）的方式通知等待的 goroutine。这样，不满足条件的 goroutine 唤醒继续向下执行，满足条件的重新进入等待序列。

结构如下：

~~~go
type Cond struct {
    noCopy noCopy
  
    // L is held while observing or changing the condition
    L Locker
  
    notify  notifyList // 通知列表
    checker copyChecker
}
func NewCond(l Locker) *Cond

//使用该函数时，可以加锁也可以不加锁，原因为该函数没有修改操作
func (c *Cond) Broadcast()//广播通知，用来唤醒所有的处于等待c状态的协程，如果没有等待的协程，该函数也不会报错。
//使用该函数时，可以加锁也可以不加锁，原因为该函数没有修改操作
func (c *Cond) Signal()//单点通知，通知单个等待c状态的协程，让它继续执行，如果此时有多个协程处于等待状态，会从等待列表中取出最开始等待的那个协程，来接收消息。
//使用该函数时，必须加锁
func (c *Cond) Wait() //等待通知，该函数在被调用之后，在没有收到Signal或者Broadcast的通知之前，协程处于阻塞状态。
~~~

##### Wait 方法

~~~go
func (c *Cond) Wait() {
    c.checker.check()
    t := runtime_notifyListAdd(&c.notify)//加入通知列表
    c.L.Unlock()//解锁 Locker
    runtime_notifyListWait(&c.notify, t)//等待通知
    c.L.Lock()//对Locker加锁
}
~~~

基本使用方法为：

~~~go
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
~~~

例子：

~~~go
// Package main provides ...
package main

import (
    "fmt"
    "sync"
    "time"
)

var count int = 4

func main() {
    ch := make(chan struct{}, 5)

    // 新建 cond
    var l sync.Mutex
    cond := sync.NewCond(&l)

    for i := 0; i < 5; i++ {
        go func(i int) {
            // 争抢互斥锁的锁定
            cond.L.Lock()
            defer func() {
                cond.L.Unlock()
                ch <- struct{}{}
            }()

            // 条件是否达成
            for count > i {
                cond.Wait()
                fmt.Printf("收到一个通知 goroutine%d\n", i)
            }
            
            fmt.Printf("goroutine%d 执行结束\n", i)
        }(i)
    }

    // 确保所有 goroutine 启动完成
    time.Sleep(time.Millisecond * 20)
    
    // 锁定一下
    fmt.Println("broadcast...")
    cond.L.Lock()
    count -= 1
    cond.Broadcast()
    cond.L.Unlock()

    time.Sleep(time.Second)
    fmt.Println("signal...")
    cond.L.Lock()
    count -= 2
    cond.Signal()
    cond.L.Unlock()

    time.Sleep(time.Second)
    fmt.Println("broadcast...")
    cond.L.Lock()
    count -= 1
    cond.Broadcast()
    cond.L.Unlock()

    for i := 0; i < 5; i++ {
        <-ch
    }
}
~~~

#### Pool 临时对象池

`sync.Pool` 可以作为临时对象保存和复用的集合。其结构为：

~~~go
type Pool struct {
    noCopy noCopy

    local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
    localSize uintptr        // size of the local array

    // New optionally specifies a function to generate
    // a value when Get would otherwise return nil.
    // It may not be changed concurrently with calls to Get.
    New func() interface{}
}

func (p *Pool) Get() interface{}
func (p *Pool) Put(x interface{})
~~~

新键 Pool 需要提供一个 New 方法，目的是当获取不到临时对象时自动创建一个（不会主动加入到 Pool 中），Get 和 Put 方法都很好理解。深入了解过 Go 的同学应该知道，Go 的重要组成结构为 M、P、G（可以通过[GMP模型分析](https://shimengfei.github.io/posts/19049.html)了解）。Pool 实际上会为每一个操作它的 goroutine 相关联的 P 都生成一个本地池。如果从本地池 Get 对象的时候，本地池没有，则会从其它的 P 本地池获取。因此，Pool 的一个特点就是：可以把由其中的对象值产生的存储压力进行分摊。

它有着以下特点：

- Pool 中的对象在仅有 Pool 有着唯一索引的情况下可能会被自动删除（取决于下一次 GC 执行的时间）。
- goroutines 协程安全，可以同时被多个协程使用。

{% note success simple %}
GC 的执行一般会使 Pool 中的对象全部移除。
{% endnote %}

适用与无状态的对象的复用，而不适用与如连接池之类的。官方例子：

~~~go
package main

import (
    "bytes"
    "io"
    "os"
    "sync"
    "time"
)

var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func timeNow() time.Time {
    return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
    // 获取临时对象，没有的话会自动创建
    b := bufPool.Get().(*bytes.Buffer)
    b.Reset()
    b.WriteString(timeNow().UTC().Format(time.RFC3339))
    b.WriteByte(' ')
    b.WriteString(key)
    b.WriteByte('=')
    b.WriteString(val)
    w.Write(b.Bytes())
    // 将临时对象放回到 Pool 中
    bufPool.Put(b)
}

func main() {
    Log(os.Stdout, "path", "/search?q=flowers")
}

打印结果：
2006-01-02T15:04:05Z path=/search?q=flowers
~~~

#### Once 仅执行一次

`sync.Once` 保证函数多次调用，只被执行一次。其结构为：

~~~go
type Once struct {
    m    Mutex
    done uint32
}

func (o *Once) Do(f func())
~~~

用 done 来记录执行次数，用 m 来保证保证仅被执行一次。只有一个 Do 方法，调用执行。

~~~go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once
    onceBody := func() {
        fmt.Println("Only once")
    }
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            once.Do(onceBody)
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}

# 打印结果
Only once
~~~

### 总结

本小结仅介绍了`sync`包下的方法的基本使用，下一节将进行相关源码分析。