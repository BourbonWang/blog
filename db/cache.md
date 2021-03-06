# 缓存雪崩、穿透、击穿

## 缓存

缓存是高并发场景下提高热点数据访问性能的一个有效手段。数据库的每秒并发量十分有限，通常将缓存放在数据库的上层，用来提升数据访问的性能，降低数据库的并发量。

缓存分为：本地缓存、分布式缓存和多级缓存。

### 本地缓存

本地缓存就是在进程的内存中进行缓存。本地缓存是内存访问，没有远程交互开销，性能最好，但是受限于单机容量，一般缓存较小且无法扩展。

### 分布式缓存

分布式缓存一般都具有良好的水平扩展能力，对较大数据量的场景也能应付自如。缺点就是需要进行远程请求，性能不如本地缓存。

### 多级缓存

为了平衡这种情况，实际业务中一般采用多级缓存。本地缓存只保存访问频率最高的部分热点数据，其他的热点数据放在分布式缓存中。

## 缓存雪崩

缓存雪崩是指，当缓存的所有key在同一时间全部失效，这期间的请求会全部流向数据库，造成相当大的压力。

### 原因

- 所有的key设置了相同的过期时间。
- 分布式缓存节点宕机。

### 解决

对于第一个原因，可以在批量存入数据时，把每个key的过期时间加个随机值，保证数据不会在同一时间大面积失效。也可以设置热点数据永不过期，需要更新时直接更新缓存即可。

## 缓存穿透

用户大量的请求不存在的key，由于这些key根本不存在，每次请求都要访问数据库，导致数据库压力过大。

### 解决

- 收到请求后先校验参数，过滤掉不合法的请求。
- 缓存和数据库中都没有取到的key，可以将 Value 写为 null、稍后重试这样的值，缓存有效时间可以设置短一点。
- 为了防止攻击用户反复用同一个 id 暴力攻击，可以在网关层 Nginx 中对每秒访问次数超出阈值的 IP 进行拉黑处理。
- 使用布隆过滤器，利用高效的数据结构和算法快速判断出 Key 是否在数据库中存在，不存在直接返回，可能存在就可以执行上面第2条。

### 布隆过滤器

Bloom Filter 用于检索一个元素是否在一个集合中。当一个元素被加入集合时，通过K个散列函数将这个元素映射成一个位数组中的K个位，把它们置为1。检索时，我们只要看看这些位是不是都是1就（大约）知道是否存在于集合了。如果这些位有任何一个0，则被检元素一定不在；如果都是1，则被检元素很可能在。

比如：使用两个哈希函数h1, h2，16bit的位数组，来记录集合（a,b,c）

```shell
假设元素对应的哈希值为：      
  h1 h2
a 1  4
b 2  6
c 3  5 
将位数组的这些位设为1：
1111110000000000
检查元素x是否在集合中，假设x的哈希值是5，10. 
检查第5和10位存在0，所以x不在集合中
检查元素y是否在集合中，假设y的哈希值是2，4.
检查第2和4位都是1，y可能在集合中，需要下一步判断。
```

Bloom Filter跟单哈希函数Bit-Map不同之处在于：Bloom Filter使用了k个哈希函数，每个字符串跟k个bit对应。从而降低了冲突的概率。

它的优点是空间效率和查询时间都远远超过一般的算法，缺点是有一定的误识别率和删除困难。

## 缓存击穿

对于一个热点数据，在失效的瞬间，大量的访问请求直接穿透到数据库。

解决方法：

- 对于热点数据，同样可以设置永不过期，然后更新数据库时同步更新缓存。
- 对同一个数据的并发请求，可以加互斥锁。保证了同一个数据同时只有一个请求访问数据库，其他的请求都阻塞，等待数据库返回。数据库返回时，同一个数据返回给所有的并发请求。

