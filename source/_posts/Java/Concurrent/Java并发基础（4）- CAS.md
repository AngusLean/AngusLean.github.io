---
title: Java并发基础（4）- CAS
date: 2017/5/4
updated: 2018/2/11
tags:
   - CAS
   - 并发
categories:
   - Programming Language
   - Java
   - 并发
---
```CAS``` 这个词在Java并发编程中还算常见，那么它到底是什么呢？
<!--more-->



在正式讨论```CAS(Compare And Swap)```策略和它是如何被```Atomic```使用之前，先看如下代码：
```

public class MyApp
{
    private volatile int count = 0;
    public void upateVisitors() 
    {
       ++count; //increment the visitors count
    }
}
```
这个代码保存访问应用的数量，很明显的是```++```操作是个三元操作，不是线程安全的。一个很明显的解决方案是：
```

public class MyApp
{
    private int count = 0;
    public synchronized void upateVisitors() 
    {
       ++count; //increment the visitors count
    }
}
```
上面的代码时线程安全的，它保证了变量的原子性和可见性。但是它有个很明显的问题： 它使用了锁-带来了较大的延迟和开销,
[参考这篇文章]( http://flex4java.blogspot.jp/2015/03/is-multi-threading-really-worth-it.html) ，这是一个非常**昂贵**的解决办法。
为了解决这个问题```atomic```非常值得介绍。 如果使用```AtomicInteger```类统计数量也可以解决上述线程安全问题：
```

public class MyApp
{
    private AtomicInteger count = new AtomicInteger(0);
    public void upateVisitors() 
    {
       count.incrementAndGet(); //increment the visitors count
    }
}
```
支持```atomic```操作的类，如```AtomicInteger,AtomicLong etc```，使用了```CAS```。
CAS没有使用锁来解决问题。它遵循如下步骤：
- 比较变量的原始值和我们已有的值
- 如果值不相同则意味着有其他线程在更改这个变量，否则就会继续往下用新的值替换原先的值。


在 ```AtomicInteger```类中如下代码：
```

public final long incrementAndGet() {  //
JDK7

    for (;;) {
        long current = get();
        long next = current + 1;
        if (compareAndSet(current, next))
          return next;
    }
}
```
与
```

public final long incrementAndGet() {  //JDK8
        return unsafe.getAndAddLong(this, valueOffset, 1L) + 1L;
}
```
它们内部到底是什么？
> 事实unsafe是JVM内部通过JIT转换为一个优化的指令序列，比如说X86处理器仅仅就是一个CPU指令```LOCK XADD```，[参考这里]( https://blogs.oracle.com/dave/atomic-fetch-and-add-vs-compare-and-swap)
也就是说，unsafe是JVM通过调用JNI的代码实现的。 


----


考虑一种情况： 如果 ```atomic```有很严重的竞态条件-许多线程想要更新同一个 ```atomic``` 变量。这时候似乎锁的性能会超过 ```atomic```,但事实上争夺 ```atomic```的性能要好过锁。 JDK引入了一个新的类：```LongAddr```, 它的介绍如下：
> 当多个线程同时更新数据时该类的性能通常要好于```AtomicInteger```,比如收集统计的时候，而不是为了细粒度的同步控制。在较低并发的情况下两个类拥有相同的特点。但是在高并发的情况下，这个类的吞吐量要明显的高一些。当然，也意味着更大的空间占用。


CAS缺点：


CAS高效的解决了原子问题，但是任然有如下三个问题：
1. ABA问题。 CAS需要在操作的时候检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值为A的改为B后又改为A,那么使用CAS检查就会认为没有发生变化，但实际上是有变化的。  解决思路就是使用版本号，使得ABA变为1A-2B-3A
从JDK1.5起atomic包引入了 AtomicStampedReference 类，这个类的```compareAndSet```方法首先检查当前引用是否是预期引用，然后检查当前标志是否是预期标志，如果全部相同，则以原子方式将该引用和标志设置为给定的新值。
2. 循环时间长，开销大。 自旋CAS如果长时间不成功，会给CPU带来非常大的开销。 如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。
3. 只能保证一个共享变量的原子操作 。当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了 AtomicReference 类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。





转载自：
https://dzone.com/articles/how-cas-compare-and-swap-java
http://zl198751.iteye.com/blog/1848575
