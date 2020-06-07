
---
title: 一种基于Redis的分布式延迟队列实现
date: 2019/10/26
updated: 2019/10/26
tags:
   - 分布式系统
   - Redis

categories:
   - 分布式系统
---
本文介绍一种基于Redis实现的延时队列，在需求相对简单的情况下可以很好的满足需求。本项目在公司内部（百万用户下）线上环境使用中，稳定性方面还算可靠。

<!--more-->

## 背景
在软件开发中，经常会遇到需要*“延迟”*处理的场景：
- 用户注册一定时间没有进行下一步，则发送相关的提醒或者营销行为
- 任务处理出错后的定时重试

等等不一而足。 而对于这种场景在单机情况下最为常用的解决方案则是：定时任务， 定时任务也就是一个会定时执行的方法，通常采用`cron`调用或者采用线程池的定时任务来处理。大部分情况下，这种方式没有什么问题。
但是，私以为至少有两个方面这种解决方案是不够优雅的：
- 调度本身依赖系统命令
任务的正确执行强依赖“运行时环境”，诚然说`cron`已经是非常成熟的技术
- 定时任务对于自身宕机、多服务器协同存在问题
对于单机的定时任务，在分布式环境下必然涉及到锁。因为每台服务器处理的任务都是相同的，如何**正确**的调度它们是一项很考验设计的活。
当然，在大数据平台下还有其他很多方案，但是这些技术所引入的”复杂性“也是成倍剧增，所以在一些不是非常”重”的场合需要一种更为轻量的方式

## 设计
现代软件系统，不论说是单机环境还是分布式环境，内存缓存都是一项不可或缺的技术，而`Redis`又在其中举足轻重。就我所经历的项目来说，可能有不依赖诸如`zookeeper`、`dubbo`、`Spring Cloud`之类的，但是，几乎没有不依赖`Redis`的。所以自然而然的考虑在`Redis`里引入延时队列。

但是，`Redis`只是一种内存缓存方案 - 基于内存。它的优势劣势可以说都很明显：
- 速度快， 大部分应用的瓶颈都不是在`Redis`上
- 容量有限,且是易失性

而对于延迟队列这种场景，对于中小型项目的需求，一般会有如下特点：
- 可以容忍延迟的“误差”
比如说延迟30分钟执行，在30分钟零几秒执行不会有太大问题
- *瞬时*的容量不大
这个瞬时指的是某个时间范围内需要延迟的数据量，Redis（或者说内存）必须可以存的下
- 可以容忍部分丢失
`Redis`系统本身可以采用集群等方式保证自己的可用性，大部分环境至少都会设置主备环境来保证可用性。

所以，基于上述”简单“的原因，基于`Redis`的延迟队列还是有一些应用场景
### 基本原理
[Redis官方](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-2-delayed-tasks/)有偏关于实现延迟队列的文章，基本上是本文的核心内容，背后的思想简单的说：
1. 采用`ZSET`来保存延迟消息，`ZSET`的`key`就是延时时间戳
2. 定时拉取上面这个`ZSET` - 通常是1秒一次（最小延迟单位）
3. 对于延迟的消息，存放到另外的队列`LIST`里
4. 多个客户端阻塞从`LIST`里获取结果

`python`代码示意如下:
增加延迟消息：
```
def execute_later(conn, queue, name, args, delay=0):
   //生成一个唯一id
    identifier = str(uuid.uuid4())
    //需要延迟的消息，包括回调名等
   item = json.dumps([identifier, queue, name, args])
//放到名为delayed的队列中，key就是需要执行的时间
    if delay > 0:
      conn.zadd('delayed:', item, time.time() + delay)
    else:
//如果延时时间小于等于0则直接执行
      conn.rpush('queue:' + queue, item)
   return identifier
```
查询到期的消息:
```
def poll_queue(conn):
   while not QUIT:
      item = conn.zrange('delayed:', 0, 0, withscores=True)
      //没有获取到延迟到期的消息
      if not item or item[0][1] > time.time():
         time.sleep(.01)
         continue

      //查询到到期的消息，则尝试获取锁后从延迟队列里移除并且增加到待执行队列
      item = item[0][0]
      identifier, queue, function, args = json.loads(item)
      locked = acquire_lock(conn, identifier)
      if not locked:
         //获取锁失败，其他线程已经做了这件事
         continue

      if conn.zrem('delayed:', item):
         conn.rpush('queue:' + queue, item)
      //释放锁
      release_lock(conn, identifier, locked)
```
### 实现考虑
上述代码给出了一个”最简“的思路，然后我们把它放在分布式环境下仔细思考，发现有如下问题需要考量：

1. 在保存消息到延迟队列时，是否需要做系统级别的隔离？ 也就是不同业务线的消息可以分开保存
2. 从阻塞队列里获取消息然后删除，这里用到了锁，能否优化？
3. 从阻塞队列里获取消息的逻辑，能不能单线程执行？
这里如果使用单线程执行，如何保证可用性？即自身挂掉如果自动切换到其他节点，其实也就是一个”保活“的概念
4. 从待执行任务队列里取数据后的回调如何处理？

从某种角度上考虑上述问题，其实这就是一个典型的”生产者-消费者“问题。多生产者、多消费者、可能还需要协调。 那么这里就可以在当前技术栈下做出合适的选择：
1. 本身是否有基于`Redis`的锁实现？ 有可以直接采用锁而不是”保活“方案
2. 本身是否已有分布式协调组件，能够较为简单的实现*选主*或者说保活的逻辑，如果有则可以采用单个任务来拉取到期任务，无锁可能更为有效
3. 合理的设计延迟队列消息之间的”隔离性“， 不同类型的延迟消息可能需要不同的待处理延迟队列已便于分布式拉取

对于这些问题，考虑到所处项目的实际需求，我毫无疑问的选择了”最为简单“的几种=_=.
[`github`地址](https://github.com/AngusLean/delay-queue4j)

简单的demo:
```

    @Test
    public void addData1() throws InterruptedException {
        delayMsgConfig.setCorePoolSize(10);
        delayMsgConfig.setMaximumPoolSize(20);
        //开始拉取延迟消息，这里还没有注册回调所以不会有任何消息会获取
        delayMsgConfig.begin();
        String system = "DELAY-ATEST1";
        CountDownLatch latch = new CountDownLatch(1);
    //注册延迟回调接口，只关心key为DELAY-ATEST1
        delayMsgConfig.addDelayCallBack(system, (uuid, message) -> {
            System.out.println("收到消息" + uuid + ", " + message);
            latch.countDown();
        });
    //添加延迟消息
        delayMsgConfig.addDelayMessage(DelayedInfoDTO.builder()
                .delayTime(Math.abs(new Random().nextLong()) % 20)
                .system(system)
                .message(system + String.format("%2d", new Random().nextInt(100)))
                .uuid(UUID.randomUUID().toString())
                .build());
        latch.await(100, TimeUnit.SECONDS);
        System.out.println("======>");
    }
```
上述代码的注册延迟消息回调函数和生成延迟消息可以分开部署，已便于最大化性能。同时，对于执行回调函数也提供了线程池的手动配置已达到最合理的资源利用率。

### 问题和改进
首先，上面项目暂时没有很好的解决”资源隔离维度“的问题，还需要更加仔细的设计
其次，对于分布式环境最好不需要加锁，这里还待改进
最后就是， `Redis`本身的特性决定了这种实现的量级不会太大。而消息队列才应该是这种需求的”最正确“解法，**消息TTL+死信队列**毫无疑问可以实现延迟的效果，而消息队列本身的解耦设计使得消费者和生产者变得更加灵活。 我觉得唯一的问题就是，团队里的成员能否*”Hold"住消息队列*，是否“知其所以然”， 因为目前通用的几个消息队列其实都是挺“重”且需要理解的点不少。
这是另外一个关于架构选型的问题了。
