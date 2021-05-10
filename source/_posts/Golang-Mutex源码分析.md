---
title: Golang Mutex源码分析
tags:
  - 技术
  - Go语言
categories: 技术
comments: true
abbrlink: e65f8f8e
date: 2021-03-29 23:12:56
cover: https://cdn.jsdelivr.net/gh/shimengfei/cdn/guidao/1.png
---
# Golang Mutex源码分析

Golang 中的Mutex 主要借助了 CAS 指令 + 自旋 + 信号量

## CAS 指令

比较并交换(compare and swap, CAS)，是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。 该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值。

### CAS 缺点

1. CAS在共享资源竞争比较激烈的时候，每个goroutine会容易处于自旋状态，影响效率，在竞争激烈的时候推荐使用锁。

2. 无法解决ABA问题

   **ABA问题**

   ***进程P1读取了一个数值A
   P1被挂起(时间片耗尽、中断等)，进程P2开始执行
   P2修改数值A为数值B，然后又修改回A
   P1被唤醒，比较后发现数值A没有变化，程序继续执行。***

## 自旋

是指当一个线程在获取锁的时候，如果锁已经被其它线程获取，那么该线程将循环等待，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环。

## 信号量

信号量的概念是计算机科学家 Dijkstra （Dijkstra算法的发明者）提出来的，广泛应用在不同的操作系统中。系统中，会给每一个进程一个信号量，代表每个进程当前的状态，未得到控制权的进程，会在特定的地方被迫停下来，等待可以继续进行的信号到来。
如果信号量是一个任意的整数，通常被称为计数信号量（Counting semaphore），或一般信号量（general semaphore）；如果信号量只有二进制的0或1，称为二进制信号量（binary semaphore）。在linux系统中，二进制信号量（binary semaphore）又称互斥锁（Mutex）
计数信号量具备两种操作动作，称为V（ signal()）与P（ wait()）（即部分参考书常称的“PV操作”）。V操作会增加信号量S的数值，P操作会减少它。
运行方式：

1. 初始化信号量，给与它一个非负数的整数值。
2. 运行P（wait()），信号量S的值将被减少。企图进入临界区的进程，需要先运行P（wait()）。当信号量S减为负值时，进程会被阻塞住，不能继续；当信号量S不为负值时，进程可以获准进入临界区。
3. 运行V（signal()），信号量S的值会被增加。结束离开临界区段的进程，将会运行V（signal()）。当信号量S不为负值时，先前被阻塞住的其他进程，将可获准进入临界区。

## 源码分析

Go 语言中`Mutex`结构体共由两个字段组成

~~~go
type Mutex struct {
  state int32
  sema  int32
}
~~~

#### state

如下图所示，`Mutex.State` 由 `waiterNum` 、`straving`、`woken`、`locked`组成。

* ***waiterNum*** 表示目前互斥锁等待队列中有多少goroutine在等待

* ***straving*** 表示目前互斥锁是否处于饥饿状态

* ***woken*** 表示目前互斥锁是否为唤醒状态

* ***locked*** 表示目前互斥锁资源是否被goroutine持有

![go-mutex-state](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/go-mutex-state.png)



#### sema

用于等待队列。

### Lock 流程分析

![image-20210329210758755](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/go-lock.png)

**Lock**

~~~go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
  /*
   通过CAS获取锁资源，获取不到则初始化当前goroutine所需要的变量
  */
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
~~~

首先使用***CAS方法***（**CompareAndSwap**）去判断是否可以直接获取锁，如果可以获取锁资源，则修改`Mutex.state`中的`locked`状态位并成功获取锁，如果获取不到，则执行lockSlow()方法。

**lockSlow**

初始化当前goroutine需要的变量，执行for循环尝试获取锁资源

~~~go
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
    //一、尝试通过自旋方式获取锁资源
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
    //二、更新互斥锁状态
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
      // 如果不是饥饿状态，则尝试更新锁的状态到new上
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
      // 如果锁处于饥饿状态或被其他goroutine持有
      // 等待队列+1
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
      // 如果当前锁为饥饿状态，并且锁资源被其他goroutine持有
      // 更新当前锁状态副本的饥饿状态的值
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
    //通过CAS设置new的值
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
       //当锁没有处在饥饿状态下，可以视作成功获取了锁
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
      // 判断是否处于等待状态
			queueLifo := waitStartTime != 0
       // 获取等待开始的时间
			if waitStartTime == 0 {
        // runtime_nanotime 实际上是一个系统调用，获取当前时间
				waitStartTime = runtime_nanotime()
			}
      /*
          如果我们没有通过 CAS 获得锁，
          会调用 sync.runtime_SemacquireMutex 
          使用信号量保证资源不会被两个 Goroutine 获取。
          sync.runtime_SemacquireMutex 
          会在方法中不断调用尝试获取锁并休眠当前 Goroutine 等待信号量的释放，
          一旦当前 Goroutine 可以获取信号量，它就会立刻返回，
      */
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
       // 当等待时间超过1ms时候，变为饥饿状态
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
        // 如果为饥饿状态的话
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
        // delta的实际值为等待的位数
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
        // 对m.state中的等待计进行ADD的原子操作
				atomic.AddInt32(&m.state, delta)
				break
			}
      // 唤醒次数为true, 自旋次数重置
			awoke = true
			iter = 0
		} else {
      // 如果CAS不成功，则获取新的state状态
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}

~~~

**一、尝试通过自旋的方式获取锁**

尝试通过自旋方式获取锁资源，自旋可以避免goroutine切换，但是消耗的资源更多，当goroutine进行自旋的时候，实际上是调用 **sync_runtime_doSpin**方法，该方法会在CPU上执行若干次PAUSE指令，也就是什么都不做，sleep一小会儿，但是会占用CPU资源。所以goroutine进入自旋的条件非常苛刻，通常需要满足以下两个条件：

1. 互斥锁只有在普通模式才能进行自旋

2. **sync_runtime_canSpin(iter)**返回true

   * 自旋次数（iter）小于4

   * ncpu > 1 也就是CPU核数大于1

   * 当前机器上有一个运行的P队列，且GOMAXPROS（可以用的处理器）大于1

     

**二、更新互斥锁状态**

主要是更新互斥锁的相关状态，详细流程可见上图蓝框部分。

**三、更新互斥锁状态后**

1. 首先尝试使用CAS设置目前new的值。

2. 如果没有成功设置则代表有新的goroutine更新了当前的锁资源，我们需要更新当前锁状态，重新进行for循环尝试获取锁。
3. 如果当前锁不处于饥饿状态以及没有被别的goroutine获取，则视为拿到锁资源。
4. 判断等待实际以及更新等待时间，调用**runtime_SemaacquireMute**x使用信号量使资源不会被两个goroutine同时获取，而当有别的goroutine释放了锁资源，则第一时间会将信号量返回给该goroutine，立即获得锁资源。
5. 当等待时间超过1ms得时候，更新饥饿状态。
6. 如果锁处于饥饿状态，且当前goroutine不属于饥饿状态，或锁位未处于饥饿状态，则退出饥饿模式
7. 如果锁不处于饥饿状态，唤醒该goroutine然后将自旋次数重置。

整个加锁过程结束。

### UnLock流程分析

![image-20210329210906862](https://cdn.jsdelivr.net/gh/shimengfei/cdn/go_img/go-unlock.png)

根据上图，分析一下`Unlock`逻辑,

1. 首先判断锁的状态，如果锁处于非饥饿且非唤醒状态，则释放锁资源，否则执行unlockSlow()

2. unlockSlow 逻辑

   1. 如果直接解锁一个没有被锁定的锁，抛出异常

   2. 判断锁是否为饥饿状态
      * 如果锁不为饥饿状态，且锁不为（锁住、唤醒、饥饿）状态的任一，直接解锁
      * 如果为上面三种情况的一种，需要唤醒在等待队列中的goroutine
      * 如果锁处于饥饿状态，直接唤醒等待队列中的goroutine.

~~~go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
  //如果直接解锁一个没有被锁定的锁，抛出异常
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
  //判断锁是否为饥饿状态
	if new&mutexStarving == 0 {
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
        //如果锁不为饥饿状态，且锁不为（锁住、唤醒、饥饿）状态的任一，直接解锁
				return
			}
			// Grab the right to wake someone.
      //如果为锁住、唤醒、饥饿三种情况的一种，需要唤醒在等待队列中的goroutine
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
    //如果锁处于饥饿状态，直接唤醒等待队列中的goroutine.
		runtime_Semrelease(&m.sema, true, 1)
	}
}
~~~

## Mutex 使用注意

**Lock/UnLock 成对使用**

**Mutex 千万不能被复制**

我们使用 Mutex 是为了不同 goroutine 之间共享某个变量, 所以需要让这个变量做到能够互斥, 不然该变量就会被互相被覆盖. Mutex 底层是由 `state` `sema` 控制的, 当 Mutex 变量被复制时, Mutex 的 state, sema 当时的状态也被复制走了, 但是由于不同 goroutine 之间的 Mutex 已经不是同一个变量了, 这样就会造成要么某个 goroutine 死锁或者不同 goroutine 共享的变量达不到互斥

~~~go
type Person struct {
 mux sync.Mutex
}

func Reduce(p1 Person) {
 fmt.Println("step...", )
 p1.mux.Lock()
 fmt.Println(p1)
 defer p1.mux.Unlock()
 fmt.Println("over...")
}

func main() {
 var p Person
 p.mux.Lock()
 go Reduce(p)
 p.mux.Unlock()
 fmt.Println(111)
 for {}
}
~~~

问题分析：

1. main Goroutine 已经给 p.mux 加了锁 , 这个时候 p.mux  的 state 的值是 mutexLocked。
2. 然后将 p.mux 复制给了 Reduce Goroutine。这个时候被复制的 p1.mux 的 state 的值也是 mutexLocked。
3. main Goroutine 虽然已经解锁了, 但是 Reduce Goroutine 跟 main Goroutine 的 mutex 已经不是同一个 mutex 了, 所以 Reduce Goroutine 就会加锁失败, 产生死锁，关键是编译器还发现不了这个 Deadlock。