# Redis 为什么快

## 性能

Redis采用的是基于内存的是单进程单线程模型的 KV 数据库，官方提供的数据是可以达到100000+的QPS。有兴趣的可以参考官方的基准程序测试《[How fast is Redis？](https://redis.io/topics/benchmarks)》

![](https://s3.ax1x.com/2021/02/07/yNlgDf.png)

横轴是连接数，纵轴是QPS。可以看出Redis的性能十分强大。但是它为什么这么快呢？

## 原因

- **基于内存。**绝大部分请求是纯粹的内存操作，非常快速。
- **数据结构简单。**Redis中的数据结构是专门进行设计的，一些常用操作的时间复杂度都是O(1)。
- **单线程。**不需要考虑锁的问题，节省了大量的加锁和阻塞的时间，避免了因死锁而导致的性能消耗。
- **使用多路复用I/O模型。**
- **自己构建的底层模型。**Redis自己构建了VM 机制 ，因为调用系统函数会浪费一定的时间去移动和请求；

## 单线程 

与多线程相比，Redis单线程的工作方式可以减少线程调度的耗时，在数据上，也不需要考虑锁，从而节省了加锁和等待锁的时间，也避免了出现死锁造成的性能消耗。

多线程真的比单线程快吗？多线程的本质是，通过CPU调度来模拟多个线程。用一个CPU做同样的内存操作，用多线程的方式会比单线程多出上下文切换的时间。对于内存系统，没有上下文切换是效率最高的。redis 用 单个CPU 绑定一块内存，然后针对这块内存的数据在一个CPU上完成读写，这时采用单线程是最佳方案。

> ### Redis is single threaded. How can I exploit multiple CPU / cores?
>
> It's not very frequent that CPU becomes your bottleneck with Redis, as usually Redis is either memory or network bound. For instance, using pipelining Redis running on an average Linux system can deliver even 1 million requests per second, so if your application mainly uses O(N) or O(log(N)) commands, it is hardly going to use too much CPU.
>
> However, to maximize CPU usage you can start multiple instances of Redis in the same box and treat them as different servers. At some point a single box may not be enough anyway, so if you want to use multiple CPUs you can start thinking of some way to shard earlier.
>
> You can find more information about using multiple Redis instances in the [Partitioning page](https://redis.io/topics/partitioning).
>
> However with Redis 4.0 we started to make Redis more threaded. For now this is limited to deleting objects in the background, and to blocking commands implemented via Redis modules. For future releases, the plan is to make Redis more and more threaded.

[官方](https://redis.io/topics/faq)表示，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。

需要注意的是，这里的单线程仅仅是处理网络请求，完整的redis是不止一个线程的。比如，redis做持久化的时候另外开辟线程执行。官方也说到，之后版本会用多线程完成一些别的操作。

**那么，什么场景适合多线程呢？**

磁盘这种慢速的操作。频繁的磁盘读写会比内存多上数量级的耗时。这时可以利用磁盘的吞吐量，用一个线程管理任务队列，达到一定数量时，统一进行磁盘读取和写入。然后多个请求线程只需将请求加入队列，然后处理线程异步的返回数据就好了。这样可以尽量的利用磁盘的性能，在磁盘面前多线程调度的耗时可以忽略。

**如何利用多核性能？**

可以在单机开多个Redis实例，因为k-v存储的数据之间没有约束，只需要分清数据在哪个实例上就好。

## I/O多路复用

多路指的是多个网络连接，复用指的是复用同一个线程。多路复用模型是利用 epoll 同时监察多个流的 I/O 事件。在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有 I/O 事件时，线程从阻塞态中唤醒，程序就会轮询发生事件的流，并且只依次顺序的处理就绪的流，这种做法就避免了大量的无用操作。

再加上 Redis 本身的事件处理模型将 epoll 中的连接、读写、关闭都转换为事件，放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理，不在网络 IO 上浪费过多的时间。

