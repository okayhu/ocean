## 什么是 Redis

Redis 是一种开源（BSD 许可）、内存中数据结构存储，用作数据库、缓存和消息代理。Redis 提供了数据结构，例如字符串、散列、列表、集合、带有范围查询的排序集合、位图、超级日志、地理空间索引和流。Redis 内置复制、Lua 脚本、LRU 驱逐、事务和不同级别的磁盘持久化，并通过 Redis Sentinel 和 Redis Cluster 自动分区提供高可用性。

## Redis 的数据类型

- Strings: 二进制安全字符串。

- Lists：按照插入顺序排序的字符串元素的集合。它们基本上是链表。
- Sets：唯一的、未排序的字符串元素的集合。

- Sorted Sets：类似于集合，但每个字符串元素都与一个浮点值相关联，称为 score。元素总是按它们的分数排序。

- Hashes：是由与值关联的字段组成的映射。字段和值都是字符串。这与 Ruby 或 Python 哈希非常相似。

- Bit arrays (or simply bitmaps)：使用特殊命令可以像处理位数组一样处理字符串值。

- HyperLogLogs：这是一种概率数据结构，用于估计集合的基数。

- Streams：Stream 是 Redis 5.0 引入的一种新数据类型，它以更抽象的方式对日志数据结构进行建模。

## Redis keys

Redis 键是二进制安全的，这意味着您可以使用任何二进制序列作为键，从像`foo`这样的字符串到 JPEG 文件的内容。**空字符串也是一个有效的键，允许的最大 key 大小为 512 MB**。

关于 key 的其他一些规则：

- 很长的键不是一个好主意。例如：1024 字节的键不仅在内存方面是个坏主意，而且因为在数据集中查找键可能需要多次昂贵的键比较。即使手头的任务是匹配一个大值的存在，对其进行散列（例如使用 SHA1）也是一个更好的主意，尤其是从内存和带宽的角度来看。
- 很短的键通常不是一个好主意。如果您可以改写 `user:1000:followers` 为 `u1000flw`，但这样作为键就没有意义了。前者更具可读性，与键对象本身和值对象使用的空间相比，增加的空间较小。虽然后者显然会消耗更少的内存，但您的工作是找到正确的平衡点。
- 尝试一种风格。例如 `object-type:id` 是个好主意，如 `user:1000` 。点或破折号常用于多词字段，如 `comment:1234:reply.to` 或 `comment:1234:reply-to` 。

## Redis Strings

```bash
> set mykey somevalue
OK
> get mykey
"somevalue"
```

SET 和 GET 命令是我们设置和检索字符串值的方式。请注意，SET 将替换已存储在键中的任何现有值。值可以是各种字符串（包括二进制数据），例如可以在值中存储 jpeg 图像。**值不能大于 512 MB**。

### [SET](https://redis.io/commands/set) 命令

> 自 1.0.0 起可用。时间复杂度 O(1)

设置 `key` 去持有字符串 `value` 。如果 `key` 已经拥有一个值，则无论其类型如何，它都会被覆盖。成功的 SET 操作将丢弃与密钥关联的任何先前的生存时间。

SET 命令支持一组修改其行为的选项：

- `EX` *seconds*：设置指定的过期时间，以秒为单位。
- `PX` *milliseconds*：设置指定的过期时间，以毫秒为单位。
- `EXAT` *timestamp-seconds*：设置密钥到期的指定 Unix 时间，以秒为单位。
- `PXAT` *timestamp-milliseconds*：设置密钥到期的指定 Unix 时间，以毫秒为单位。
- `NX`：仅在不存在的情况下设置密钥。
- `XX`：仅设置已存在的密钥。
- `KEEPTTL`：保留与键关联的生存时间。
- `GET`：返回存储在 key 中的旧字符串，如果 key 不存在，则返回 nil。如果 key 中存储的值不是字符串，则返回错误并中止 SET。

> 由于 SET 命令选项可以替换 [SETNX](https://redis.io/commands/setnx)、[SETEX](https://redis.io/commands/setex)、[PSETEX](https://redis.io/commands/psetex)、[GETSET](https://redis.io/commands/getset)，因此在未来的 Redis 版本中，这些命令可能会被弃用并最终被删除。

```bash
> set counter 100
OK
> incr counter
(integer) 101
> incr counter
(integer) 102
> incrby counter 50
(integer) 152
```




