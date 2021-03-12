# 进程同步

## 临界区

在多个进程同时访问一个资源时，容易发生并发问题。比如，进程1和进程2同时对公用的计数器counter进行自加操作`counter++`。直观上可能认为这样一条语句不会产生并发问题。但是自加操作在CPU层面是这样的：

```c
register1=counter  
register1=register1+1
counter=register1
```

进程在执行上面语句的过程中，很可能因为上下文切换被打断，从而打断了counter的自加操作。

```c
P1：register1=counter          // register1 = 5
P1：register1=register1+1      // register1 = 6
// 上下文切换, 执行P2
P2：register2=counter          // register2 = 5
P2：register2=register2+1      // register2 = 6
P2：counter=register2          // counter = 6
// 上下文切换，执行P1
P1：counter=register1          // counter = 6
```

在上面的例子中，假设counter初值为5，P1、P2分别对其进行自加操作，预期的结果是counter = 7. 但由于进程间的竞争，出现了错误的结果。多个进程同时访问和操作相同的数据，而执行的结果取决于访问的特定顺序，这种情况称为竞争条件。为了避免这种情况，要确保同时只有一个进程访问共享的资源，进程之间需要一些手段来同步。

每个进程都会有这样一部分关键代码，会访问到共享的资源，这样的关键代码段称为临界区。当一个进程在临界区里执行时，不允许其他的进程进入其临界区。这样就避免了竞争条件。上面例子中 `counter++`就是临界区代码，当进程1在执行这句时，进程2须等待进程1执行完才能执行。

临界区的实现方案需要在进入临界区之前和之后执行某些操作，来保护临界区。为了保证并发的正确执行，对临界区的方案有下面的要求：

- 互斥。任何两个进程不能同时在临界区里。
- 临界区外的进程不能阻塞其他进程进入其临界区。
- 有限等待。不能让进程无限的等待进入临界区。

## 互斥锁

很容易想到用互斥锁来保护临界区，进程在进入临界区之前获得锁，在离开后释放锁。

```c
bool lock;
void enter() {    // 进入临界区前执行的函数,
    while (lock);
    lock = true
}

void leave() {    // 离开临界区执行的函数
	lock = false   
}
```

但是这种方法有一个问题，当一个进程读出lock = false，打算获得锁。而恰好在它修改lock = true 之前，另一个进程被调度，也读到了lock = false，并且将它设置为true。调度回第一个进程时，它同样设置lock = true。这样就导致了两个进程同时处于临界区。因此，互斥锁的方案要求获得和释放锁都是原子操作，这通常用于硬件实现。

## Perterson 算法

```c
int turn;              // 现在轮到谁
bool flag[2];          // 进程是否准备好进入临界区，初值为false，假设有2个进程

void enter(int i) {    // 进程i进入临界区前执行的函数,
    flag[i] = true;    
    j = 1 - i          // 另一个进程号
    turn = j;           
    while (flag[j] && turn == j);  // 如果另一个进程在临界区里，等待
}

void leave(int i) {    // 进程i离开临界区执行的函数
	flag[i] = false;   
}
```

当进程0想进入临界区，调用`enter(0)`, 设置turn = 1, 在循环判断时 flag[1] = false，循环退出函数返回，进程0很容易进入临界区。之后如果进程1想进入临界区，调用`enter(1)`，设置 turn = 0，在循环判断时，条件为真，会一直等待。直到进程0退出时，将flag[0] 设为 false，进程1才能退出循环进入临界区。

设想进程0和进程1同时想进入临界区。它们都将flag置为true。假设进程0先置turn，turn会保持后执行的赋值结果： turn = 0. 然后两个进程在循环判断时，因为turn，先执行的进程0会退出循环，后执行的进程1会进入等待。这样也可以实现互斥。

设置turn为另一个进程的意义是，将进入权先交给其他进程，如果其他进程也想进入，turn就指明了能进入临界区的进程；确保其他进程都不想进入的时候，自己才进入临界区。

### 忙等待

Perterson的解决方案是正确的，但缺点是需要连续的等待循环，称为忙等待。这种互斥锁也称为自旋锁，因为锁在等待中会一直循环，直到变得可用。这种方法不仅会浪费CPU时间，还会出现优先级反转问题。

## 优先级反转

当较高优先级的进程需要访问较低优先级进程正在访问的资源，高优先级进程必须要等待低优先级进程完成，如果此时低优先级进程被调度，高优先级进程等待资源的时间将增加。

比如，三个进程L、M、H，它们的优先级L<M<H. 当进程L获取了一个资源后，进程H想获得资源必须等待L执行完。如果此时进程M被调度执行，抢占了进程L。那么，具有较低优先级的进程M影响了高优先级进程H的等待时间。

这个问题可以通过优先级继承来解决。低优先级进程访问高优先级进程所需要的资源时，或继承高优先级，访问资源完成后，优先级将恢复为原来值。在上面的例子中，进程L继承了进程H的优先级，从而防止进程M抢占其执行。当进程L结束资源访问后，将优先级恢复为原来的较低值。由于此时资源可用，下一个被调度的是进程H，而不是M。

## 信号量

如果我们想允许一定数量的进程同时访问公共资源，这时候就需要使用信号量。信号量是一个整数，代表了允许同时在临界区的进程数量。信号量通过两个原子操作`signal()` 和`wait()`来增减。

信号量可以是任何整数。当信号量为负，代表了当前等待的进程数量。当信号量初值为1时，保证了同时仅有1个进程访问临界区，称为二元信号量，和互斥锁的功能相同。

### 实现

```c
typedef struct{
    int value;
    struct process *list;
}semaphore;

void wait(semaphore *S){
    S->value--;
    if (S->value < 0){
        add this process to S->list;
        block();
    }
}

void signal(semaphore *S){
    S->value++;
    if (S->value <= 0){
        remove a process P from S->list;
        wakeup(P);
    }
}
```

为了解决忙等待问题，当进程需要等待时，可以将自己挂起，放入信号量的等待队列中，进程状态切换到waiting，CPU会调度其他进程执行。当信号量允许有进程进入临界区时，等待队列中的进程会被唤醒。

### 同步

信号量还可以用于进程间同步。将信号量初始值设为0，然后在两个进程间插入如下语句：

```
P1：            P2：
...             wait(sem)
signal(sem)     ...
```

P1执行到signal时，由于信号量为0，P1阻塞。当P2执行到wait时，信号量+1，将P1唤醒。然后P1和P2就可以同步运行了。

## 经典问题

### 生产者-消费者问题

The Bounded-Buffer Problem。有一个固定大小的缓冲区，生产者进程不停的往里写入数据，直到满。消费者不停的从缓冲区里拿走数据，直到空。保证同时只有一个进程访问缓冲区。

```c
int N;                 // 缓冲区大小
semaphore mutex = 1;   // 互斥锁，保护缓冲区
semaphore empty = N;   // 记录缓冲区空槽数量
semaphore full = 0;    // 记录缓冲区满槽数量

void producer() {
    while (true) {
        wait(&empty);
        wait(&mutex);
        /* add an item to the buffer */
        add_buffer(item);
        signal(&mutex);
        signal(&full);
    }
}

void consumer() {
    while (true) {
        wait(&full);
        wait(&mutex);
        /* remove an item from buffer */
        int item = remove_buffer();
        signal(&mutex);
        signal(&empty);
    }
}
```

### 读者-写者问题

对一个共享资源，允许多个进程同时读，但同时只有一个进程可以执行写操作，并且读操作与写操作之间互斥。这时因为读操作并不会修改数据，就不会发生并发问题；而写操作需要隔离。这也是读写锁的实现。

```c
semaphore rwmutex = 1;  // 写锁，保证读写互斥
semaphore mutex = 1;    // 保护readcount 
int readcount = 0;      // 读者数量

void writer() {
    while (true) {
        wait(&rwmutex);   // 写者需要等待写锁
        /* writing */
        ...
        signal(&rwmutex);
    }
}

void reader() {
    while (true) {
        wait(&mutex);
        readcount++;
        if (readcount == 1)    // 当有读者时，加写锁，让写者阻塞
            wait(&rwmutex);
        signal(&mutex);
        /* reading */
        ...
        wait(&mutex);
        readcount--;
        if (readcount == 0)    // 当没有读者，释放写锁，此时写者可以写
            signal(&rwmutex);
        signal(&mutex);
    }
}
```

### 哲学家用餐问题

有5个哲学家和5根筷子，哲学家可以拿起他左右两边的两根筷子来吃饭，一根筷子同时只能被一个人使用。

![](https://s3.ax1x.com/2021/02/16/yc7YsH.png)

哲学家是相互独立的5个进程，筷子是5个共享资源，当一个哲学家在使用筷子时，需要对筷子加锁。

```c
void philosopher (int i) {
    while (true) {
        wait(&chopsticks[i]);
        wait(&chopsticks[(i+1)%5]);
        eat();
        signal(&chopsticks[(i+1)%5]);
        signal(&chopsticks[i]);    
    }
}
```

很容易想到这种实现。但存在死锁的问题：当5个进程同时拿起左边的筷子，没有进程能够拿起右边的筷子，所以进程持有的筷子也无法释放，所有进程都将阻塞。一些解决方案：

- 只允许4个进程同时拿起筷子，这样就不会出现循环等待造成的死锁。可以使用信号量来计数。

- 将取筷子操作作为临界区，同时只允许一个哲学家取筷子。这样，当一个哲学家所需要的筷子被别人使用时，他将阻塞在临界区内直到得到筷子，而其他的哲学家无法进入临界区。

  ```c
  void philosopher (int i) {
      while (true) {
          wait(mutex);
          wait(&chopsticks[i]);
          wait(&chopsticks[(i+1)%5]);
          signal(mutex);
          eat();
          signal(&chopsticks[(i+1)%5]);
          signal(&chopsticks[i]);    
      }
  }
  ```

- 奇数序号的哲学家先拿起左边的筷子，而偶数序号的哲学家先拿起右边的筷子，也可以解决循环等待的问题。

  ```c
  void philosopher (int i) {
      while (true) {
          if (i % 2 == 1) {
              wait(&chopsticks[i]);
              wait(&chopsticks[(i+1)%5]);
              eat();
              signal(&chopsticks[(i+1)%5]);
              signal(&chopsticks[i]);    
          }else{
              wait(&chopsticks[(i+1)%5]);
              wait(&chopsticks[i]);
              eat();
              signal(&chopsticks[i]);    
    			signal(&chopsticks[(i+1)%5]);
          }
          
      }
  }
  ```

  