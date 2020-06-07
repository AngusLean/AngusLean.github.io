---
title: Java并发基础（1.1）从双重检查锁模式说起
date: 2018/3/28
updated: 2018/3/28
tags:
categories:
   - Programming Language
   - Java
   - 并发
---
双重检查锁模式，`double-checked locking`在设计模式以及并发编程中可以说是一个非常“出名”的概念。很多的例子或者书上都会有如下的代码:


<!--more-->



```java
private static Something instance = null;
public static Something getInstance() {
  if (instance == null) {
    synchronized (this) {
      if (instance == null)
        instance = new Something();
    }
  }
  return instance;
}
```
这段代码看起来非常清晰：
1. 判断实例是否为空
2. 如果为空则对当前类加锁
3. 再次判断实例是否为空（上面的锁保证了可见性）
4. 再次判断还是为空的话则创建该类的一个实例，并且赋值到实例上


看起来几乎没有问题-- 但是实际上是有问题的。
最大的问题是， ** 在多线程环境下， 初始化一个`Something`实例与把这个实例赋值给`instance`变量可能被编译器或者CPU缓存等给重排序**， 这就导致了线程可能会看到一个**部分初始化**的对象- 这当然不对。 （这个错误也有算法证明及其他[例子](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)）


上面提到，**重排序**明显是问题的关键,并且这个词我们在其他地方也听过多遍。 那么这是Java特有的问题还是整个并发编程都有的问题呢？在较为底层的指令（汇编/CPU指令）上，到底是怎样一个场景呢？  我们从一个来自于《thr art of multiprocessor programming》的叫做` Peterson lock`的算法的例子说起（这是一个使用于2个线程的锁的算法）。


```

class Peterson
{
private:
    // 通过thread ID, 0 or 1定位
    bool _interested[2];
    // 谁拥有优先权
    int _victim;
public:
    Peterson()
    {
        _victim = 0;
        _interested[0] = false;
        _interested[1] = false;
    }
    void lock()
    {
        // threadID 是线程私有的
        // 初始化为0或者1
        int me = threadID;
        int he = 1 - me;
        _interested[me] = true;
        _victim = me;
        while (_interested[he] && _victim == me)
            continue;
    }
    void unlock()
    {
        int me = threadID;
        _interested[me] = false;
    }
}



```
大概介绍下这个算法。当线程A想要获得一个锁的时候，把A的`_interested`槽（即数组中对应的位置）设为`true`（或者叫做设为对这个锁有兴趣），线程A是这个槽的唯一写入者，它也把`_victim`设为指向自己。然后开始检查是否有其他线程对这个锁刚好持有（另外一个线程同样会设置 `_interested`槽 ）并且这个时候自己感兴趣（ `_victim`为自己 ），那么就会一直循环（`continue`）知道其他线程设置 `_interested` 槽为false，同时我还是这个锁感兴趣的话就可以结束这个`lock`调用。 而在析构的时候， 设置 `_interested` 槽为false表示自己已经不需要。


然后我们简化问题，不考虑说` _interested `这个数组，转为两个变量: `zeroWants,oneWants`来代表数组的两项，也就是如下的变量:
```
zeroWants = false;
oneWants = false;
victim = 0;
```
然后他们的高级语言表示：
Thread 0 :
```
zeroWants = true;   
victim = 0;
while (oneWants && victim == 0)
 continue;
// critical code
zeroWants = false;
```


Thread 1:
```
oneWants = true;
victim = 1;
while (zeroWants && victim == 1)
 continue;
// critical code
oneWants = false;
```


他们的伪汇编语言表示:
Thread 0:
```
store(zeroWants, 1)
store(victim, 0)
r0 = load(oneWants)
r1 = load(victim)
```


Thread 1:


```
store(oneWants, 1)
store(victim, 1)
r0 = load(zeroWants)
r1 = load(victim)
```


可以看到，他们做的事情是几乎一样的，保存2个变量，然后加载2个变量。 但是，处理器是可以自由移动对于变量` oneWants`的读到对于变量` zeroWants`写之前，可以自由移动对于变量` zeroWants `的读到对于变量` oneWants`的写之前去的， 因为他们之间没有上下文的依赖关系 - 重排序并不会改变语义。 所以说， 他们可能会被重排序成这个样子:
Thread 0:
```
r0 = load(oneWants)
store(zeroWants, 1)
store(victim, 0)
r1 = load(victim)
```


Thread 1:


```
r0 = load(zeroWants)
store(oneWants, 1)
store(victim, 1)
r1 = load(victim)
```


初始化情况下，变量` zeroWants, oneWants `都被初始化为0， 所以说`r1和r2`可能都是0. 那么上面C代码的循环也就永远不会执行，这个算法也就无法正常运行了。
一种解决这个问题的方法是内存屏障（memory fence），内存屏障会强制CPU最终执行的是正确的顺序，它可以被放到同一个线程中` zeroWants `的保存和` oneWants `的读之间的任意位置，或者2个线程中 ` oneWants `的写和 ` zeroWants `的读之间。 这样可以保证2个线程不会同时看到两个变量都是为空，进而可以保证这个算法的正确性。 

