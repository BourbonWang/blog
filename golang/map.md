# Go map

## map

map使用哈希表所谓底层实现，一个哈希表里有多个哈希桶 bucket，每个bucket里保存着有相同哈希值（低位相同）的一个或多个键值对。查看 `runtime/map.go`：

```go
type hmap struct {
	count     int        // 当前元素个数
	B         uint8      // bucket个数，有 2^B 个
	hash0     uint32     // 哈希种子
	buckets    unsafe.Pointer // 指针指向bucket数组
	oldbuckets unsafe.Pointer // 旧的bucket数组，用于扩容
    ...
    
}
```

map的数据存放到bucket里，计算哈希后，用哈希值的低位来选择对应的bucket。比如当 B = 2 时，有2 ^ 2 = 4个bucket，数据结构如下图：

![](https://oscimg.oschina.net/oscnet/a680d221f582e35a1ea8f52616d5c4ca7d2.jpg)

## bucket

一个bucket里存放的键的哈希值的低位相同。每个bucket里能存放8个键值对。

```go
type bmap struct {
    tophash [8]uint8 // 每个元素hash值的高8位
    // 接下来是8个key、8个value，但是我们不能直接看到；
    // data byte[1]
    // 再接下来是hash冲突发生时，下一个溢出桶的地址
    // overflow *bmap
}
```

- tophash 是个长度为8的数组，存放每个键的哈希值的高位，以方便后续匹配。以往的哈希表实现中，每个bucket可能只存一个键值对，但这里能存8个，因为检查tophash的8个元素也很快。例如：B = 2, 有4个bucket，假设 hash(key1) = 10, hash(key2) = 22, 它们都将存放到第二个 bucket 里，但它们哈希值的高位不同，这样就可以从tophash里定位到数据了。
- data存放 key-value 数据。存放顺序是key/key/key/...value/value/value，如此存放是为了节省字节对齐带来的空间浪费。
- overflow 指针指向的是下一个bucket。因为当出现哈希冲突时，一个bucket只能存8个键值对，这是不够的，需要额外的bucket，然后用链表的形式连接起来。

![](https://oscimg.oschina.net/oscnet/1aa15e9f3d78b3e66dacbead2990f78aa08.jpg)

可以把bucket看做链表节点，整个链表代表了具有相同哈希值低位的数据。这些链表的表头存在map的buckets数组中。如果哈希冲突很严重，overflow 的bucket很多，那么查找效率会很慢，因为在链表中是线性查找。

## 扩容

负载因子用于衡量一个哈希表冲突情况，负载因子 = 键数量 / bucket数量。

哈希表需要将负载因子控制在合适的大小，超过其阀值需要将键值对重新组织。哈希因子过小，说明空间利用率低；哈希因子过大，说明冲突严重，存取效率低。触发扩容的条件有两个：

1. 负载因子 > 6.5时，也即平均每个bucket存储的键值对达到6.5个。
2. overflow数量 > 2^15时，也即overflow数量超过32768时。

扩容后的新buckets数组大小是原来的两倍，然后旧bucket数据搬迁到新的bucket。考虑到如果map存储了数以亿计的key-value，一次性搬迁将会造成比较大的延时，Go采用逐步搬迁策略，即每次访问map时都会触发一次搬迁，每次搬迁2个键值对。过程如下：

- hmap数据结构中oldbuckets指向原来的bucket数组，而buckets指向了新申请的bucket数组。新数组的长度是原来的2倍
- 新的bucket数组的前半部分，与旧数组的哈希相同；后半部分是增加的哈希值的bucket。比如，旧的B = 3，有8个bucket，存放的哈希值低位 0 - 7。扩容后，有16个bucket，存放的哈希值 0 - 15。原来在0 - 7的bucket中的数据，可能会移动到后面的bucket中。
- 所以检查每个键值对的哈希值低位 x , 判断 x < 2 ^ B，如果是，则该键值对仍在原来的索引x中；如果不是，该键值对需要移动到索引 x + 2 ^ B的桶中。
- 每个溢出的bucket也需要遍历。在频繁的新建删除中，旧buckets的存储可能非常松散，有很多个溢出桶但每个的空闲空间却很多。扩容的过程也相当于对这些空间进行了压缩。遍历后，溢出的bucket数量能够减少。
- 当oldbuckets都转移完，释放内存，扩容完成。

## 存取过程

插入元素：

1. 根据key值计算出哈希值
2. 哈希值低位与 2 ^ B 取模，确定bucket下标
3. 去对应的bucket里，遍历tophash寻找哈希值高位
4. 当前bucket没找到，寻找下一个overflow的bucket
5. 找到了，更新值；没找到，插入值

查询元素：

1. 根据key值计算出哈希值
2. 哈希值低位与 2 ^ B 取模，确定bucket下标
3. 如果当前处于扩容状态，优先从oldbuckets查找
4. 去对应的bucket里，遍历tophash寻找哈希值高位
5. 当前bucket没找到，寻找下一个overflow的bucket
6. 直到找到值，返回

## 参考链接

[0]: https://my.oschina.net/renhc/blog/2208417?nocache=1539143037904
[1]: https://blog.csdn.net/luolianxi/article/details/105371079#t1

