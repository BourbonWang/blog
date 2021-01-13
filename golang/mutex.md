# 互斥锁 Mutex 与 读写锁 RWMutex

## 互斥锁

在操作系统中，当多线程并发运行时，可能会访问或修改到共享的代码。比如下面这个例子，多个goroutine同时修改 n 的值：

```go
n := 0
add := func() {
    n = n + 1
}
for i := 0; i < 100; i++ {
    go add()
}
time.Sleep(time.Second)
fmt.Println(n)
```

最终的输出很有可能不是100. 这是因为 `n = n + 1`在底层被解释为取值、加操作、赋值的3个操作。由于上下文切换，某个线程在执行这3个操作的过程中会被中断，另一个线程开始执行，从而导致了数据不一致。

这些共享资源的代码部分称为临界区（Critical Section）。应当保证临界区内代码的原子性。操作系统中使用信号量来保护临界区。对于上面这个例子，我们可以用二元信号量来保护临界区，即互斥锁。线程得到了资源，则对资源加锁，使用结束后释放锁。资源被加锁，其他线程将无法得到该资源，直到锁被释放。这样就保证了同时仅有1个线程正在访问资源。

## Mutex

### 使用

Go 提供了 `sync.Mutex`来实现这个功能。Mutex只有两个方法：Lock() 和 Unlock()，分别代表了加锁和解锁。对于上面的例子，使用Mutex 来保护临界区：

```go
n := 0
mu := sync.Mutex{}
add := func() {
	n = n + 1
}
for i := 0; i < 100; i++ {
    mu.Lock()
    go add()
    mu.Unlock()
}
```

### 状态

查看源码，mutex的结构：

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

state 是状态码，包含了4项内容：

- Locked：表示是否上锁，上锁为1 未上锁为0
- Woken：表示是否被唤醒，唤醒为1 未唤醒为0
- Starving：表示是否为饥饿模式，饥饿模式为1 非饥饿模式为0
- waiter：剩余的29位则为等待的goroutine数量

### 自旋

加锁时，如果当前Locked位为1，说明该锁当前由其他协程持有，尝试加锁的协程并不是马上转入阻塞，而是会持续的探测Locked位是否变为0，这个过程即为自旋过程。自旋时间很短（go设计为自旋4次），但如果在自旋过程中发现锁已被释放，那么协程可以立即获取锁。此时即便有协程被唤醒也无法获取锁，只能再次阻塞。

自旋的好处是，当加锁失败时不必立即转入阻塞，有一定机会获取到锁，这样可以避免协程的切换。

自旋的坏处是，如果自旋过程中获得锁，则马上执行该 goroutine。如果永远在自旋模式中，那么之前阻塞的goroutine 则很难获得锁，这样一来一些 goroutine 则会被阻塞时间过长。

### 普通模式 / 饥饿模式

go 对 mutex 的分配设计了两种模式。

在普通模式下，等待者以 FIFO 的顺序排队来获取锁，但被唤醒的等待者发现并没有获取到 mutex，并且还要与新到达的 goroutine 们竞争 mutex 的所有权。

在饥饿模式下，mutex 的所有权直接从对 mutex 执行解锁的 goroutine 传递给等待队列前面的等待者。新到达的 goroutine 们不要尝试去获取 mutex，即使它看起来是在解锁状态，也不要试图自旋。

## 读写锁

互斥锁的本质是当一个线程得到资源的时候，其他线程都不能访问。这样在资源同步，避免竞争的同时也降低了程序的并发性能。程序由原来的并行执行变成了串行执行。

其实，当我们对一个数据只做读操作的话，是不存在资源竞争的问题的。因为数据是不变的，不管多少个线程同时读取，都能得到同样的数据。所以问题不是出在读上，主要是写。要保证同时仅有一个线程在修改数据。所以真正的互斥应该是读和写、写和写之间，多个读者间没有互斥的必要。

因此，衍生出了读写锁。读写锁可以让多个读操作并发，但是对于写操作是完全互斥的。也就是说，当一个线程进行写操作的时候，其他线程既不能进行读操作，也不能进行写操作。

## RWMutex

### 原理

操作系统中，可以使用信号量实现读写锁，也可以使用二元信号量即互斥锁实现。Go提供了读写锁`sync.RWMutex`，定义如下：

```go
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // 写锁需要等待读锁释放的信号量
	readerSem   uint32 // 读锁需要等待写锁释放的信号量
	readerCount int32  // 读锁后面挂起了多少个写锁申请
	readerWait  int32  // 已释放了多少个读锁
}
```

可以看出RWMutex是基于Mutex实现的，在其基础上增加了读写的信号量。

读锁与读锁兼容，读锁与写锁互斥，写锁与写锁互斥，只有在锁释放后才可以继续申请互斥的锁：

- 可以同时申请多个读锁
- 有读锁时申请写锁将阻塞
- 只要有写锁，后续申请读锁和写锁都将阻塞

### 使用

提供了以下几个方法：

```go
//申请和释放写锁
func (rw *RWMutex) Lock()
func (rw *RWMutex) Unlock()
//申请和释放读锁
func (rw *RWMutex) RLock()
func (rw *RWMutex) RUnlock()
//返回一个实现Lock()和Unlock()的接口
func (rw *RWMutex) RLocker() Locker
```

如果不存在写锁，则Unlock()引发panic，如果不存在读锁，则RUnlock()引发panic

下面给出例子

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type data struct {
	N   int
	RWM sync.RWMutex
}

func read(d *data, t time.Time) {
	d.RWM.RLock()
	time.Sleep(1 * time.Second)
	fmt.Printf("reader: n = %d, %s\n", d.N, time.Now().Sub(t).String())
	d.RWM.RUnlock()
}

func write(d *data, t time.Time) {
	d.RWM.Lock()
	time.Sleep(3 * time.Second)
	d.N++
	fmt.Printf("writer: n = %d, %s \n", d.N, time.Now().Sub(t).String())
	d.RWM.Unlock()
}

func main() {
	t := time.Now()
	var d data
	for i := 0; i < 10; i++ {
		go read(&d, t)
	}
	for i := 0; i < 5; i++ {
		go write(&d, t)
	}
    
	time.Sleep(20 * time.Second)
}
```

创建了10个读线程，每个睡眠1秒；5个写线程，睡眠3秒。记录读者和写者的开始时间。输出如下：

```
reader: n = 0, 1.000319542s
reader: n = 0, 1.000435996s
reader: n = 0, 1.000242171s
reader: n = 0, 1.00048857s
reader: n = 0, 1.000339988s
reader: n = 0, 1.000224101s
reader: n = 0, 1.000995623s
writer: n = 1, 4.001359749s 
reader: n = 1, 5.001534292s
reader: n = 1, 5.001578192s
reader: n = 1, 5.001575223s
writer: n = 2, 8.001839462s 
writer: n = 3, 11.00235864s 
writer: n = 4, 14.002692469s 
writer: n = 5, 17.002956352s 
```

可以看出，由于可以同时读，多个读者间并没有发生阻塞。而写者由于锁机制，存在阻塞，延时3秒。