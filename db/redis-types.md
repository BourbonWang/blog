# Redis 数据类型

Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

## String

string 是 redis 最基本的类型，是key - value存储。string 类型是二进制安全的，可以接受任何数据，一般存放整数、浮点数、字符串，也可以是jpg图片或json。string 类型的值最大能存储 512MB。

### 应用场景

- **自增：**使用自增（incr）统计网站访问数量，当前在线人数等。redis的操作都是原子性的，不会出现并发冲突。

- **存储对象：**存储 json 或其他对象格式化的字符串。这种场景下推荐使用 hash 数据类型。

  ```shell
  set user:id:1 '[{"id":1,"name":"aa","email":"74326524974@qq.com"},{"id":2,"name":"bb","email":"23o978564@qq.com"}]'
  ```

- **存储MySQL中字段值**：把 key 设计为 表名：主键名：主键值：字段名

  ```shell
  set user:id:1:email 34876534057@qq.com
  ```

### 常用命令

| 命令                         | 描述                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| set key value                | 设置指定 key 的值                                            |
| GET key                      | 获取指定 key 的值                                            |
| INCR key                     | 将 key 中储存的数字值增一                                    |
| DECR key                     | 将 key 中储存的数字值减一                                    |
| MGET key1 [key2…]            | 获取所有(一个或多个)给定 key 的值                            |
| SETEX key seconds value      | 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位) |
| SETNX key value              | 只有在 key 不存在时设置 key 的值                             |
| STRLEN key                   | 返回 key 所储存的字符串值的长度                              |
| MSET key value [key value …] | 同时设置一个或多个 key-value 对。                            |
| GETRANGE key start end       | 返回 key 中字符串值的子字符                                  |
| GETSET key value             | 将给定 key 的值设为 value ，并返回 key 的旧值(old value)     |

## Hash

hash类型的key是一个唯一值，value是一个hashmap。使用hash类型可以将多个key-value存储到一个key里。

hash天然的适合存储对象和结构化数据。如果使用string存储对象，需要对序列转换和解析，在修改时需要将整个对象反序列化，再修改，然后序列化存储；如果对分别存储对象的多个key-value，容易产生太多的key，浪费内存；使用hash就不没有这些问题。

比如：存储 `{{field1: value1},{field2: value2}}`: 

```shell
hmset key_name field1 value1 field2 value2 
```

### 常用操作

| 命令                                     | 描述                                                |
| ---------------------------------------- | --------------------------------------------------- |
| HSET key field value                     | 将哈希表 key 中的字段 field 的值设为 value          |
| HGET key field                           | 获取存储在哈希表中指定字段的值                      |
| HDEL key field2 [field2]                 | 删除一个或多个哈希表字段                            |
| HGETALL key                              | 获取在哈希表中指定 key 的所有字段和值               |
| HEXISTS key field                        | 查看哈希表 key 中，指定的字段是否存在               |
| HMGET key field1 [field2]                | 获取所有给定字段的值                                |
| HMSET key field1 value1 [field2 value2 ] | 同时将多个 field-value (域-值)对设置到哈希表 key 中 |
| HSETNX key field value                   | 只有在字段 field 不存在时，设置哈希表字段的值       |

 ## List

List 是按照插入顺序排序的字符串链表，可以在头部和尾部插入新的元素。

List使用双向链表实现，两端添加元素的时间复杂度为 O(1)。在头部和尾部插入数据时，性能会非常高，不受链表长度的影响；但如果在链表中插入数据，性能就会越来越差。插入元素时，如果 key 不存在，redis 会为该 key 创建一个新的链表，如果链表中所有的元素都被移除，该 key 也会从 redis 中移除。

### 应用场景

- 队列和栈：list的链表实现非常适合队列和栈。可以在其上实现消息队列，生产者将任务push进list，消费者用pop将任务取出。
- 最新内容：因为 list 结构的数据查询两端附近的数据性能非常好，所以适合一些需要获取最新数据的场景，比如新闻类应用的最近新闻。

### 常用操作

| 命令                      | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| LPUSH key value1 [value2] | 将一个或多个值插入到列表头部                                 |
| LPOP key                  | 移出并获取列表的第一个元素                                   |
| LLEN key                  | 获取列表长度                                                 |
| LRANGE key start stop     | 获取列表指定范围内的元素                                     |
| LREM key count value      | 移除列表元素                                                 |
| LTRIM key start stop      | 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。 |
| RPUSH key value1 [value2] | 在列表中添加一个或多个值                                     |
| RPOP key                  | 移除并获取列表最后一个元素                                   |
| RPUSHX key value          | 为已存在的列表添加值                                         |

## Set

Set 数据类型是一个没有重复元素的无序集合，可以对 set 类型的数据进行添加、删除、判断是否存在等操作，时间复杂度是 O(1) 
set 集合不允许数据重复，如果添加的数据在 set 中已经存在，将只保留一份。
set 类型提供了多个 set 之间的聚合运算，如求交并补差，这些操作在 redis 内部完成，效率很高。

### 应用场景

- 利用交集共同好友列表；或者共同好友多于k时，推荐好友
- 利用唯一性，统计网站访问量里的独立IP

### 常用操作

| 命令                       | 描述                                  |
| -------------------------- | ------------------------------------- |
| SADD key member1 [member2] | 向集合添加一个或多个成员              |
| SCARD key                  | 获取集合的成员数                      |
| SDIFF key1 [key2]          | 返回给定所有集合的差集                |
| SINTER key1 [key2]         | 返回给定所有集合的交集                |
| SISMEMBER key member       | 判断 member 元素是否是集合 key 的成员 |
| SMEMBERS key               | 返回集合中的所有成员                  |
| SREM key member1 [member2] | 移除集合中一个或多个成员              |
| SUNION key1 [key2]         | 返回所有给定集合的并集                |

## Sorted Set

有序集合。在 set 的基础上给集合中每个元素关联了一个分数，往有序集合中插入数据时会自动根据这个分数排序。集合中的元素仍然不能重复，但分数可以重复。

sorted set的内部使用HashMap和跳跃表(SkipList)来保证数据的存储和有序，HashMap里放的是成员到score的映射，而跳跃表里存放的是所有的成员，排序依据是HashMap里存的score,使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单。

### 使用场景

- 排行榜：积分榜，好友亲密度榜等
- 带权重的消息队列

### 常用操作

| 命令                                           | 描述                                                         |
| ---------------------------------------------- | ------------------------------------------------------------ |
| ZADD key score1 member1 [score2 member2]       | 向有序集合添加一个或多个成员，或者更新已存在成员的分数       |
| ZCARD key                                      | 获取有序集合的成员数                                         |
| ZCOUNT key min max                             | 计算在有序集合中指定区间分数的成员数                         |
| ZINCRBY key increment member                   | 有序集合中对指定成员的分数加上增量 increment                 |
| ZINTERSTORE destination numkeys key [key …]    | 计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中 |
| ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT] | 通过分数返回有序集合指定区间内的成员                         |
| ZRANK key member                               | 返回有序集合中指定成员的索引                                 |
| ZREM key member [member …]                     | 移除有序集合中的一个或多个成员                               |
| ZREMRANGEBYSCORE key min max                   | 移除有序集合中给定的分数区间的所有成员                       |
| ZSCORE key member                              | 返回有序集中，成员的分数值                                   |
| ZREVRANGEBYSCORE key max min [WITHSCORES]      | 返回有序集中指定分数区间内的成员，分数从高到低排序           |
| ZUNIONSTORE destination numkeys key [key …]    | 计算给定的一个或多个有序集的并集，并存储在新的 key 中        |

## key通用操作

| 命令                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| DEL key                | 该命令用于在 key 存在时删除 key。                            |
| EXPIRE key             | seconds 为给定 key 设置过期时间。                            |
| EXPIREAT key timestamp | EXPIREAT 的作用和 EXPIRE 类似，都用于为 key 设置过期时间。 不同在于 EXPIREAT 命令接受的时间参数是 UNIX 时间戳(unix timestamp)。 |
| EXISTS key             | 检查给定 key 是否存在                                        |
| KEYS pattern           | 查找所有符合给定模式( pattern)的 key                         |
| MOVE key db            | 将当前数据库的 key 移动到给定的数据库 db 当中                |
| PERSIST key            | 移除 key 的过期时间，key 将持久保持                          |
| PTTL key               | 以毫秒为单位返回 key 的剩余的过期时间                        |
| TTL key                | 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)   |
| TYPE key               | 返回 key 所储存的值的类型                                    |

