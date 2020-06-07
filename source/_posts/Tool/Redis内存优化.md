---
title: Redis内存优化(译)
date: 2019/12/16
updated: 2019/12/16
tags:
   - Redis
categories:
   - Tool
   - Redis

---


本文翻译自Redis的[官方文档](https://redis.io/topics/memory-optimization)，对部分内容已引用的格式添加自己的说明。 这篇文章可以对Redis的内存优化有一个“总览”上的概念，
而具体的**why**则需要深入到Redis的源码去探究。

<!--more-->

## 对小数据类型的特殊编码
从Redis 2.2开始，许多数据类型被优化为占用更少的空间而不是固定空间。`Hashes, Lists, 只包含整数的Sets, Sorted Set `,当它们的元素
数量小于某个配置时，或者到达某个最大元素大小配置时，便会被编码为一个接近10倍空间节省的数据结构（平均下来也有5倍）

> 这里的数据结构指的是ziplist结构，参见源码

对于用户视角来说这个是完全透明的，这是一个CPU/内存的取舍，同时，也可以通过如下配置更改这个最大元素数量/最大元素大小
```
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
set-max-intset-entries 512
```


如果一个特殊编码的值超过了配置的最大大小，Redis会自动转换为正常的结构。对于小数据量来说这个速度非常快，但是如果你改变了上述
配置那么建议做个`benchmarks`来检查这个转换时间


## 使用32位实例
Redis编译为32位，由于指针更小所以每个key会占用更少的空间,但是这也会限制Redis使用的最大内存为4G.为了编译32位的Redis可以使用命令
`make 32bit`, RDB文件和AOF文件兼容32位和64位的实例，也兼容大端和小端，所以你可以在两者自由切换而没有其他问题
> 参考[维基百科][](https://zh.wikipedia.org/zh-hant/32%E4%BD%8D%E)

## Bit和byte层级的操作
Redis 2.2引入了新的Bit和byte操作命令 `GETRANGE,SETRANGE,GETBIT,SETBIT`,使用这些命令你可以把Redis的`string`看做是一个随机访问的字符
数组。比如说对于某个系统来说，用户可以用递增的数字来表达，你就可以使用一个`BitMap`来存储用户的信息，设置某个位以做标记、清除某个
位以做删除. 换句话说，在一个Redis实例中，1亿个用户才占用12M的内存. 你也可以使用`GETRANGE,SETRANGE`命令为每个用户设置一个字节的信
息，这只是一个例子，但它说明了使用这些新的数据结构可以使用很小的空间表达很大的范围。

## 尽可能的使用`Hashes`
小的Hashes被压缩到非常小的空间，所以你应该每次都尝试使用`Hashes`来存储你的数据。比如你用一个对象来代表网站上的一个用户，使用一个
包含所有`key`的`hash`而不是`name, username, email`都分别存储。下一节是原因

## 使用`Hashes`来达到极高的内存使用效率
先陈述一个事实：少量的key会比使用一个包含少量字段的hash要占用更多的内存
> 就是说在量级不大的情况下，key value这种结构会比hash占用更多的内存

这里实质上是个`trick`，理论上来说，我们应该使用常量时间复杂度的数据结构来保证查询在O(1)的时间，比如`hash table`
但是，很多时候一个Hash表只包含少量的字段，当字段较少时我们可以把它编码为一个O(N)的数据结构中，比如一个长度最先的线性key-value
数组。当N比较小的时候，`HGET,HSET`仍然可以认为是`O(1)`的，当`hash table`增长到一定大小时又会自动转换为真正的`hash table`(可以在`redis.conf`
中配置)
这不仅仅是从时间复杂度上来说非常有优势，同时，在在CPU缓存上线性数据结构也比hash要更好
> 参考CSAPP中《编写良好局部性的程序》相关章节，不过个人觉得这个可以作为概念理解，实际编程中真的就是一个trick,大部分场景下可读性
> 都更重要

然而，`Hash`的字段和值都不是一个完整的`Redis`对象(此处指和其他的对比，比如string）,`hash`字段不能有个过期时间，并且仅仅只能包括`string`，
但是这其实恰恰就是`Redis`的`hash`数据结构api设计原则-简单重于features,所以说多余的功能不被支持-比如字段的过期时间。
所以说，`Hash`更加的节省内存，用`Hash`来表达对象或者其他有关联关系的字段是非常具有优势的。但是如果我们有一个纯粹的key-value逻辑时怎么
处理？
考虑我们使用`Redis`来作为许多小对象的缓存，他们可以是`JSON`编码，可以是小的`HTML`片段，简单的`key-boolean`结构等等，基本上来说就是一些小
的`string->string`的映射关系。
比如我们想缓存如下的编号:
```
object:102393
object:1234
object:5
``````

我们可以这样做：每次我们使用`SET`来保存一个新的键，我们把这个实际上的键切分为两部分：一个部分作为`Hash`的名字，也就是`key`,一部分作为
键，而值依然保存真是的值。比如对于`object:1234`,我们把它表达为:
```
名字为`object:12`的Hash结构
键为34, 值也就是真实的值
```

所以，我们把真实的key的除了最后两位作为一个单独的`Hash`，后面两位作为其中的一个键。这样，这个`Hash`里面我们就可以存储100个键-当然，
这是一个内存和CPU上的妥协优化。
还有一个非常重要的点需要注意：使用这种范式每个`Hash`都会有100左右的键值对而不用考虑值的具体类型。这是因为我们的值总是已数字结尾
而不是一个随机字符串-某种程度上，最后的数字是一个内部的预先分片(``pre-sharding``)的形式.

对于小数字怎么处理，比如`object:22`？我们就使用`object`作为`Hash`的名字即可。
使用这种方式我们节省了多少内存？这里使用一段`ruby`程序统计:
```
require 'rubygems'
require 'redis'

UseOptimization = true

def hash_get_key_field(key)
    s = key.split(":")
    if s[1].length > 2
        {:key => s[0]+":"+s[1][0..-3], :field => s[1][-2..-1]}
    else
        {:key => s[0]+":", :field => s[1]}
    end
end

def hash_set(r,key,value)
    kf = hash_get_key_field(key)
    r.hset(kf[:key],kf[:field],value)
end

def hash_get(r,key,value)
    kf = hash_get_key_field(key)
    r.hget(kf[:key],kf[:field],value)
end

r = Redis.new
(0..100000).each{|id|
    key = "object:#{id}"
    if UseOptimization
        hash_set(r,key,"val")
    else
        r.set(key,"val")
    end
}
``````

这是一个64位Redis 2.2上的数据，我们看到：
- `UseOptimization`开启，使用1.7M内存
- `UseOptimization`关闭，使用11M内存

这是一个我认为让Redis在key-value存储达到最大存储效率的方式。

注意，上述功能必须在有以下配置的情况下才有效:
```
hash-max-zipmap-entries 256
hash-max-zipmap-value 1024

```
每当`Hash`超出配置的元素数量或者元素大小时，Redis都会把这个`Hash`自动转换为一个真正的`hash table`，也就是失去了内存压缩。
你可能会问，为什么不把这些直接实现在`key`空间从而让调用者不需要关心。这里有两个原因：第一就是我们想使得这个取舍(`trade off`变得明显，因为
它综合考虑了很多因素：CPU，内存，最大元素大小),第二个原因是默认的`key`需要支持其他有趣的特性比如过期、LRU等等，这使的它
不适合来做这些优化。
`Redis`的方式是用户必须理解已达到最大的内存效率，知道`Redis`内部是如何处理这些数据以及会表现出怎样的行为.


## 内存分配
为了存储用户的key,`Redis`尽可能的分配达到`maxmemory`设置的内存（还可能轻微的超出）
这些附加属性可以通过配置文件设置或者[CONFIG SET]()https://redis.io/commands/config-set, 查看[使用Redis作为LRU缓存]()https://redis.io/topics/lru-cache
,但是，以下的也还是必须要注意:
- Redis并不总是会把删除了的key的内存返回给操作系统。对于Redis来说这不是什么特殊的事情但是大多数`malloc()`实现是。比如说你存储了一个5G的对象
然后移除部分内容只剩下2G，实际使用内存(RSS,实际使用的物理内存，包含共享库占用的)可能还是5G。这是由于底层的`allocator`不能轻易的释放内存，比如说经常性的被删除的
那部分与还存在的key处于相同的页
> 这里的"页"指的是虚拟内存中的页，参考操作系统相关文章
- 前面一点意味者你需要使用你的**峰值内存使用量**来预估内存，如果你的负载时不时需要10G内存，即使说大部分时间都只需要5G,你也需要提供10G的内存
- 然而,`allocator`也足够智能可以复用空闲的块(`chunks`),当你从5G里面释放了2G的块后，当你再次增加更多的`key`时，你会看到实际占用物理内存(RSS)还是保持原样没有
增加，直到达到2G的数据量,因为分配器尝试复用之前释放的2G空间.
- 由于这些原因，当你的内存占用在峰值的时候远远大于当前值的时候内存碎片率就不是那么可靠,碎片是根据物理内存当前实际占用来计算的。由于RSS反应了峰值
内存，当由于大量key/value被删除时实际使用内存很低但是占用依然很高，内存碎片率(RSS/mem_used)也就很大.

如果没有设置`maxmemory`,Redis会尽可能的、不断的分配内存直到耗尽所有空闲内存。所以通常建议配置这个参数，你可能还想要为了`noeviction`配置`maxmemory-policy`
,它使得Redis在达到限制时对于写请求返回一个` out of memory`,这可能会在应用程序里报错但是避免了由于内存饥饿耗死机器。


> PS：本文对于Redis的内存优化几乎是总览性的，但是为什么说ziplist/zipmap占用小以及说具体的内存满了的策略则需要另外学习。对本文的内容了然于心在开发中
> 也可以做到有的放矢的设计。
