# Redis 特性

------

## bitmap

bitmap就是用最小的单位bit来代表key，通过设置对应bit的值0或者1，表示某个key对应的值或者状态。redis中bit映射被限制在512MB之内，所以最大是2^32位。bitmap有以下优点：

- 基于最小的单位bit进行存储，所以非常省空间。
- 设置时候时间复杂度O(1)、读取时候时间复杂度O(n)，操作是非常快的。
- 二进制数据的存储，进行相关位运算的时候非常快。
- 方便扩容

### 命令

- `getbit key offset` 对key所存储的字符串值，获取指定偏移量上的位（bit）  
- `setbit key offset value` 对key所存储的字符串值，设置指定偏移量上的位0或1. 返回该位原来的值. offset从0开始，可以超过bitmap的长度。 
- `bitcount key [start end]` 获取位图指定范围中位值为1的个数. start和end的单位为byte。如果不指定start与end，则取所有  
- `bitop op destKey key1 [key2...]` 做多个BitMap的and、or、not、xor操作, 并将结果保存在destKey中  
- `bitpos key tartgetBit [start end]` 计算位图指定范围第一个偏移量对应的的值等于targetBit的位置. 找不到返回-1, start与end没有设置，则取全部

### 应用

1. **用户在线状态：**由于用户ID是唯一的，所以可以用bitmap的每一位代表一个用户ID。在线为1，不在线为0。3亿用户只需要36MB的空间。
2. **用户权限**：在应用中的用户权限可能很多，并且经常随着业务修改，比如：是否关闭弹幕，是否接收推送，添加好友是否需要验证等。这些不适合作为固定的属性添加到用户信息表中。可以为每个权限用一个bitmap管理所有用户，这样增加和删除权限时，只需要增加和删除对应的bitmap即可。

------

## HyperLogLog

Redis HyperLogLog 是用来做基数统计的数据结构。基数就是不重复元素的数量。比如数据集{1，2，2，2，3，4，5，5}，它的无重复元素的集合是{1，2，3，4，5}，基数就为5. 

### 特点

HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的。每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

### 命令

- `PFADD key element [element...]` 将元素加入HyperLogLog中。
- `PFCOUNT key [key...]` 返回给定key的基数。多个key会返回基数之和
- `PFMERGE destkey srckey [srckey...]` 将多个HyperLogLog合并成一个。 

### 应用

基数统计可以用在统计网站的独立访客量。比如对商品链接进行访客量统计，需要为每个商品都建立一个数据结构，然后将用户标识存入数据结构，然后统计。

传统的数据结构如B树，可以得到不错的插入和查询代价。但在空间上，如果每个商品的访客量巨大，而商品数量也非常多，那么建立B树将占用巨大的内存，假设每个B树占用1M,  10万个B树就要100G。如果将B树存入磁盘中，很难进行合并操作。如果使用bitmap作为集合，bitmap可以高效的查询和合并。但bitmap的内存占用与基数的上限有关，假如要计算上限为1千万的基数，则需要1M字节的bitmap，10万个商品同样要100G，并且每个bitmap的空间是固定分配的。所以bitmap也不合适。采用HyperLogLog则可以很好的解决需求。

------

## GEO

Redis GEO 主要用于存储地理位置信息，并且可以进行相关计算。

https://www.runoob.com/redis/redis-geo.html

------

## 发布订阅

Redis 发布订阅 (pub/sub) 是一种消息通信模式：发布者发送消息，订阅者接收消息。

当一个客户端通过 PUBLISH 命令向订阅者发送信息的时候，我们称这个客户端为发布者(publisher)。
而当一个客户端使用 SUBSCRIBE 或者 PSUBSCRIBE命令接收信息的时候，我们称这个客户端为订阅者(subscriber)。一个客户端可以订阅任意数量的channel。
为了解耦发布者和订阅者之间的关系，Redis 使用了 channel 作为两者的中介—— 发布者将信息直接发布给 channel ，而 channel 负责将信息发送给适当的订阅者。发布者和订阅者之间没有相互关系，也不知道对方的存在。

![](https://www.runoob.com/wp-content/uploads/2014/11/pubsub2.png)

### 应用

- 任务通知：系统向channel中发布任务，用户作为订阅者从channel中获取任务。不同的用户群体通过不同的channel获得各自的任务。
- 参数刷新：当前端的参数需要更新时，比如轮播图、广告等，可以使用发布订阅来通知各个前端。

------

## 事务

### 概念

Redis 事务的本质是一组命令的集合。事务支持一次性执行多个命令，一个事务中所有命令都会被序列化。在事务执行过程，会按照顺序执行队列中的命令。其他客户端提交的命令请求不会插入到事务的命令序列中。

- **Redis事务没有隔离级别的概念：**批量操作在发送 EXEC 命令前被放入队列缓存，并不会被实际执行，也就不存在事务内的查询要看到事务里的更新，事务外查询不能看到。
- **Redis不保证原子性：**单条命令是原子性执行的，但事务不保证原子性，且没有回滚。事务中任意命令执行失败，其余的命令仍会被执行。
- **Redis中的事务并没有关系型数据库中的事务回滚(rollback)功能**。使用者必须自行处理，不过这也使得Redis的事务简洁快速。

### 命令

- **MULTI**: 标记一个事务的开始。在这下面输入的命令都会被加入事务队列。

- **EXEC**：执行事务队列的命令。

- **DISCARD**：取消事务，放弃队列里的所有命令。

- **WATCH**：事务中的命令要全部执行完之后才能获取每个命令的结果，但是如果一个事务中的命令B依赖于他上一个命令A的结果，就需要使用WATCH来监视key。如果在事务执行之前，被监视的key被其他命令改动，则事务被打断 （ 类似乐观锁 ）

  当事务执行EXEC后，无论事务使用执行成功， WATCH 对变量的监控都将被取消。所以当事务执行失败后，需重新执行WATCH命令对变量进行监控，并开启新的事务进行操作。

- **UNWATCH**：取消 WATCH 命令对所有 key 的监视。

### 过程

- 开始事务

  ```Redis
  MULTI
  ```

- 命令入队：输入的命令会加入队列中，不会立即执行。

  ```
  set k1 v1 
  set k2 v2
  ...
  ```

  Redis会检查语法错误，如果其中的命令有错误，事务中的所有命令都不被执行。

- 执行事务

  ```
  EXEC
  ```

  如果在命令运行过程中出现运行错误，比如用GET获取一个hash类型的键值，这种错误在命令执行之前是无法发现的，事务中这样的命令仍然会被执行，其他的命令也会被执行。

  如果WATCH的值被修改，事务将会被打断然后取消。
