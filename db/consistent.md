# 数据库和缓存的双写一致性

## 前言

使用缓存时，都是先读取缓存，如果有数据就直接返回，如果缓存无数据则访问数据库，再将数据写入缓存。但是当数据需要更新时，如何保证数据库和缓存的数据一致性？

如果设置了缓存过期时间，那么完全不需要考虑这个问题，因为缓存过期后自然会从数据库拿到新数据。而如果没有设置过期时间，或者需要尽快完成更新操作，那么数据更新的过程可以分成两个操作：删除缓存和更新数据库。下面讨论二者的先后顺序。

## 先删除缓存再更新数据库

假设：有请求A进行数据更新，请求B同时读取数据，有以下情景：

1. 请求A删除缓存
2. 请求B读缓存发现不存在，查询数据库得到旧值
3. 请求B将旧值写入缓存
4. 请求A将新值写入数据库

这种情景下出现了不一致的情况，缓存里存了脏数据。可以发现，在删除缓存和更新数据库之间进行的读请求，都会读到脏数据。

可以通过延时双删解决。在删除缓存后，更新数据库，然后再次删除缓存，就可以删掉出现的脏数据了。既然这样，为什么不先更新数据库再删除缓存呢？

## 先更新数据库再删除缓存

观察发现，脏数据来自于请求数据库，然后对缓存执行的写操作。当旧数据还在缓存中的时候，不会发生访问数据库的情况，这时，先更新数据库再删除缓存的方式，可以避免存到脏数据。

但如果旧数据已过期不在缓存中，考虑以下场景：

1. 请求B查询数据库得到旧值
2. 请求A将新值写入数据库
3. 请求A删除缓存
4. 请求B将旧值写入缓存

这种情况仍然存到了脏数据。但观察发现，只有完全按照上面的顺序才会出现不一致。事实上，数据库的读操作的耗时远小于写操作，请求B最先读数据库又最后写缓存，这种情况几乎不可能发生。

## 总结

因此，选择更新数据库再删除缓存的方案。如果考虑到删除缓存失败的情况，这时必然会出现数据不一致。可以增加重试机制，将删除任务加入队列，然后从队列获取要删除的key，直到删除成功。