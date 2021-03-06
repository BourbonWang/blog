# TCP 详解

TCP是运输层的可靠运输协议。HTTP，HTTPS，SSH，Telnet，FTP等应用层协议都是基于TCP。

## 特点

- TCP是面向连接的。因为在一个进程可以开始向另一个进程发送数据之前，这两个进程必须先握手，即必须相互发送一些特殊的报文，以确定数据传输所需的参数。
- TCP提供全双工服务。如果进程A与进程B存在TCP连接，那么应用层数据就可以在两个进程间双向流通。
- TCP连接是点对点的。即单个发送方与单个接收方间的连接。TCP用（源IP地址，源端口号，目的IP地址，目的端口号）四元组来唯一标识进程。
- TCP是面向字节流的。对于每个连接，TCP会建立发送缓冲区和接收缓冲区。上层应用会将数据放入缓冲区，TCP会按MSS（最大报文段长）分割成一个个报文，在合适时机发送出去。接收数据的时候，应用也是从缓冲区读取数据。因此TCP的读与写不需要完全匹配。
- TCP通过报文段检验和，确认应答，超时重传，流量控制，拥塞控制等机制提供可靠传输。

## 报文

![TCP报头](https://farm1.staticflickr.com/792/27194088468_4cb0141fc8_b.jpg)

- 32位序号: 序号是该报文段首字节的编号。

- 32位确认号：TCP是全双工的，在发送数据的同时也在接收对方的数据。主机A填充进报文段的确认号是主机A期望从B收到的下一字节的序号。序号和确认号是保证TCP可靠传输的关键。

- 4位首部长度：从首部到数据部分的偏移量，一般TCP首部长度为20字节

- 6位标志位

  URG: 标识紧急指针是否有效
  ACK: 标识确认序号是否有效
  PSH: 用来提示接收端应用程序立刻将数据从tcp缓冲区读走
  RST: 要求重新建立连接. 我们把含有RST标识的报文称为**复位报文段**
  SYN: 请求建立连接. 我们把含有SYN标识的报文称为**同步报文段**
  FIN: 通知对端, 本端即将关闭. 我们把含有FIN标识的报文称为**结束报文段**

- 16位窗口：用于流量控制，指示接收方愿意接收的字节数量

- 16位检验和：校验和不光包含TCP首部, 也包含TCP数据部分

## 可靠数据传输

### 确认应答机制

TCP使用序号和确认号实现可靠传输。当A向B发送数据时，携带序号Seq Num为该报文段的首字节的编号；确认号Ack Num为期望对方下次发送报文的首字节编号，同时也告诉对方Ack Num之前的数据都已接收。从直观上，发送的每个报文都应当得到一个专门的ACK回应。实际上，双方往往是在互相发送各自的数据，然后各自回应对方。因此可以在发送自己的数据的时候，顺便带上回应对方的ACK。

![](https://s3.ax1x.com/2021/01/21/s5CToD.png)

### 滑动窗口

窗口大小指的是无需等待确认应答就可以继续发送数据的最大值. ![](https://img-blog.csdn.net/20180620002804100?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM2NjI5Njk2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 发送方将维护一个滑动窗口。窗口内的报文不需要等待ACK应答，直接发送。

对于接收方：如果一个序号n的分组被正确的接收，且按序（即上次交付给上层的是序号n-1的分组），则为序号n的分组回应ACK，然后交付给上层。对于正确但无序的分组，TCP对将其暂时缓存，应答ACK仍为最近按序正确接收的分组序号。

因此，如果发送方收到分组k的应答ACK，说明k和k之前的分组都已成功接收。发送方可以将被确认过的数据从缓冲区删掉，窗口向前移动，继续发送其他的报文。

### 超时重传

![](https://s3.ax1x.com/2021/01/21/s4jM7j.png)

发送的数据报文或应答的ACK报文可能在网络中丢失。当发送报文一段时间后仍未收到确认ACK，TCP将重新发送该报文，然后重置定时器。

![](https://s3.ax1x.com/2021/01/21/s4XWYq.png)

对于连续发送的报文，如果第一个报文触发了超时重传，而在新的超时时间之前收到了第二个报文ACK，第二个报文将不会重传。

### 快速重传

超时重传的问题是超时周期可能较长。当一个分组丢失时，发送方可能要等待很久才重传分组，从而增加端到端的时延。事实上，发送方往往一次发送大量的分组，当有分组丢失时，会引起大量相同的冗余ACK。如果TCP发送方收到了超过3个冗余的ACK，它就认为这之后的分组发生丢失，TCP将进行快速重传。

![](https://s3.ax1x.com/2021/01/21/s4jcjO.png)

## 建立和结束连接

TCP采用三次握手来创建连接，四次挥手来断开连接。详细请见另一篇文章: [TCP三次握手和四次挥手](network/three-way-handshake.md)

## 流量控制

前面说到，双方都为该连接设置了接收缓存。当接收到正确的报文后，它就将数据放入缓存。上层的应用进程会从缓存中读取数据，但没必要数据刚到达就读取，应用进程甚至可能过很长时间后才读取。如果数据读取十分缓慢，而发送方发送的数据太多太快，会导致接收缓存溢出。

为此，TCP提供了流量控制。流量控制是保证了发送速率和接收方的读速率相匹配。接收方将剩余缓存大小填入TCP报头的“窗口”字段，用于告诉发送方，自己还有多少缓存空间。窗口越大，说明接收方的吞吐量越高。由于TCP是全双工的，双方都各自维护一个接收窗口。

当接收方的缓冲区快满了，会将更小的窗口通知发送方，发送方会减慢自己的发送速度。当窗口为0时，发送方不再发送数据，但会定期的发送只有一个字节的报文段，用于探测接收方的窗口。

## 延迟应答

如果接收数据后立即进行ACK应答，这时返回的窗口可能比较小。实际上，接收方的处理数据速度可能很快，稍等一段时间就可以得到更大的缓冲区，返回更大的窗口。窗口越大, 网络吞吐量就越大, 传输效率就越高。TCP的目标是在保证网络不拥堵的情况下尽量提高传输效率。但延迟应答也有限制：

- 数量限制: 每隔N个包就应答一次
- 时间限制: 超过最大延迟时间就应答一次

## 拥塞控制

数据的丢失一般是在网络拥塞时由于路由器缓存溢出引起的。因此，分组重传是网络拥塞的征兆，但不能解决网络拥塞问题。TCP需要另一些机制在面临网络拥塞时遏制发送方。TCP拥塞控制的基本思想是，当出现丢包事件（收到3个冗余ACK）时，让发送方通过减小拥塞窗口的大小来降低发送速率。TCP的拥塞控制算法包括：1. 加性增，乘性减（AIMD）; 2. 慢启动

### 加性增，乘性减（AIMD）

每发生一次丢包事件，发送方就将当前的窗口大小减半。但不能降到低于一个MSS（最大报文段长）。

当收到前面数据的ACK时，就可以认为当前网络没有拥塞，可以考虑增加发送速率。这种情况下，TCP每次收到一个ACK确认后就把窗口增加，每个往返时延内拥塞窗口增加1个MSS。

总而言之，TCP感受到网络无拥塞就加性的增加发送速率，感受到网络拥塞时就乘性的减小发送速率。因而称为加性增，乘性减（Additive-Increase，Multiplicative-Decrease）算法。

### 慢启动

在TCP连接刚开始时，初始的拥塞窗口被设为1个MSS，而实际可用的带宽往往比这大很多。仅仅线性的增加发送速率，可能要很长时间才能达到最大的速率。因此，TCP发送方在初始阶段并不是线性的增加发送速率，而是指数级的。直到发生第一个丢包事件（或到达阈值），窗口大小减为一半，然后才会加性增。这个指数级的增加发送速率的过程称为慢启动。

每当一个报文被确认后，窗口都增加1个MSS，从而使发送速率指数增长。比如：初始时只有1个MSS，发送一个报文。收到确认后，窗口增加1个MSS，然后就可以发出两个报文。这两个报文被确认后，窗口扩大到了4个MSS。因此在慢启动阶段，每过一个RTT，窗口都增加一倍。

为了方便窗口减半和控制慢启动的窗口上限，发送方记录了窗口的阈值。初始时设定为一个较大的值。慢启动阶段达到阈值后将开始加性增。然后每次增加窗口时，阈值都会随之增加。当出现丢包或超时事件时，阈值减半，即新的窗口大小。

事实上，TCP对超时事件的反应与丢包事件（收到3个冗余ACK）有所不同。当出现超时事件，TCP将阈值减半，然后将窗口直接减为1个MSS，然后重新开始慢启动阶段，直到达到减半后的阈值。TCP对丢包事件与超时事件采取不同策略是因为，当收到3个冗余ACK，仅代表一些报文丢失，而其他一些报文能够收到，TCP会尽可能的试探网络上能利用的带宽。

![](https://s3.ax1x.com/2021/01/22/sIbRMD.png)

