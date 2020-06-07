
---
title: Java并发基础（5）并发概念与synchronized现细节
date: 2019/3/28
updated: 2019/3/28
tags:
categories:
   - Programming Language
   - Java
   - 并发
---
在[Java并发基础（2）-Synchronized](http://anguslean.cn/2017/04/17/Java/Concurrent/Java%E5%B9%B6%E5%8F%91%E5%9F%BA%E7%A1%80%EF%BC%882%EF%BC%89-Synchronized/)一文里对`synchronized`关键字的使用细节上做了一个比较详细的说明，那么这个“强大”的关键字是如何实现的呢？

<!--more-->

## 并发的几个基本问题
在具体说`synchronized`的实现之前，必须对它所解决的问题有个基本的概念：
在多线程环境中，并发上的问题在编译器、JIT、操作系统、CPU(乱序执行、多核)的共同作用下变得较为复杂。但是归根结底，其实是以下几个问题：
### 可见性
可见性指的是，某线程对某个内存区域的更改对于其他线程是否可见的问题。具体来说，在CPU的层次，某个核心上具体线程对某个内存位置的更改可能仅仅是刷新到L2/L1缓存上而没有实际的刷新到内存上去，同时，其他几个核心自己内部的缓存可能对该内存位置也有相应的缓存数据，**当某个线程对内存区域的某个变量做了更改，如何保证其他所有线程（CPU核心）都知道并且可以看见这个最新值**
### 并发写/并发读
这个问题其实与上面的如出一辙，多个线程同时对某个内存区域做更改，那么该内存区域最后的值是什么？ 取最后到达的还是什么策略？ 而每个线程看到的是本线程的更改结果还是内存位置的实际值？
解决方案很简单- 不允许**同时**写，一个线程在做可能与其他线程互相冲突的操作时其他线程必须**回避**（也就是互斥）,保证"同一时刻"只有一个线程在做这个操作。
### 顺序性（`reordering`）
现代CPU几乎都有强大的乱序执行功能，对于某些指令序列通过一系列的分析，可以**推断出可以乱序执行**-也就是代码里看到的顺序逻辑实际上被“同时执行”，这个同时执行既可能是CPU的时间片上的逻辑执行也可能是分配到不同核心山的执行。 CPU保证这些指令被同时执行不改变指令的原有语义， 这个特性很大程度上提高了指令的执行效率，在单线程时代也没有任何问题。
但对于多线程就可能有问题了，在单线程里前后没有依赖关系的代码多线程环境下就可能有问题。

## 操作系统的方案
对于互斥性问题，本质上是要求“某一段代码”同一时间只有一个线程在执行， 也被称为“临界区(`critical section`)”。满足临界区的方案会有以下几个特点：
- 不会同时有超过1个线程处于临界区
- 不对CPU速度和数量（包括虚拟的核心）做任何假设
- 临界区外的进程互补阻塞
- 进程不能无限期的等待临界区

在操作系统上，实现临界区通常有几种方式：
### 屏蔽中断
在单处理器系统中，最简单的办法是进入临界区过后立马屏蔽所有的中断，离开之前再打开， 屏蔽过后，时钟中断也被屏蔽，这样CPU就不会被切换到其他线程。
但是现代处理器不可能只有一个核心，而屏蔽中断只对调用中断的这个核心有效，其他CPU仍可以继续运行和访问共享内存，所以实质上这个方案不合理，也没有被大量使用

### 软件模拟-锁变量
考虑有一个共享变量（锁变量）存在，它的初始值为0. 线程在进入临界区之前检查这个值是否为0，如果是则讲它设为1并且进入临界区，否则等待直到该值变为0.
该方案的问题是，如果一个进程再发现它的值为0，在把它设为1之前有另外一个线程进入了临界区也把它设为了1，当第一个进程运行时它也把它设为1. 这样就会有两个线程同时处于临界区。

### 软件模拟-自旋锁
使用一个变量turn,来标识哪个线程进入临界区并检查和更新共享内存。
```
//线程1
while(true){
    while(turn != 0); //忙等待
    critical_region(); //临界区逻辑
    turn = 1; //切换到下一个线程
    noncritical_region(); //非临界区逻辑
}
//线程2
while(true){
    while(turn != 1); //忙等待
    critical_region(); //临界区逻辑
    turn = 0; //切换到下一个线程
    noncritical_region(); //非临界区逻辑
}

```
自旋的方案对于竞争不太激烈的情况和线程之前速度相差不大的来说比较合适，否则对于资源的浪费和等待时间是不可控的。同时，非临界区的代码实质上也是被临界区的代码（即使是其他线程）给阻塞了

### *Peterson*解法
该方法不做过多介绍，可以自行查阅下
### `TSL`指令-硬件方法
这是一个具备原子性的硬件指令-测试并加锁(`test and set lock`),它将一个内存字lock读到寄存器RX中，然后在该内存地址存一个非0值。 读和写不可分割，即它们是原子性的，该指令结束之前其他处理器均不允许访问该内存字。执行TSL的CPU将锁住内存总线已禁止其他CPU在期间访问该内存。
注意的是锁住总线不同于中断，它是可以锁住其他CPU的。这是非常不同于方案1的一点。
该方案使用共享变量lock来协调进入临界区，当lock为0时，任何进程都都可以使用`TSL`把lock设为1并读写共享内存，当操作结束时再把lock设为1。对于临界区则是：
1. 将lock原来的值复制到寄存器中并设为1
2. 比较寄存器中的值是否是原来的值(0)，如果不是则说明以前已经被加过锁了则回到开始重新执行
3. 如果是0，则说明加锁成功，进入临界区成功
4. 清楚这个锁只需要把这个值为0即可

```
enter_region:
    TSL REGISTER,LOCK  //读取并设为1
    CMP REGISTER, #0 //测试是否已经被锁
    JNE enter_region //循环
    RET //进入临界区成功
```
在`intel x86`CPU中底层使用的是`XCHG`指令，但同上并无实际区别。该方案较为完整

 这个方式同上面几种方式一样，都是在无法进入临界区的时候采用**忙等待**的方式。这种方法不仅浪费了了CPU时间还可能因为进程优先级的问题导致出现[优先级反转问题](https://baike.baidu.com/item/%E4%BC%98%E5%85%88%E7%BA%A7%E7%BF%BB%E8%BD%AC/4945202?fr=aladdin). 另外一种进程间同步的方式则是在无法进入临界区时阻塞。直到另外的线程把它唤醒



###
## JVM的方案
首先上述的并发上的问题在计算机科学中早已出现，并且有了一系列的成熟的解决方案。JVM在这方面并无太多的实质上创新机制，但由于虚拟机平台的存在，还是相较于传统的C/C++等语言在语法糖上提供了更甜的特性。其中的典型便是`synchronize`关键字和`JUC`包里的`AQS`类。
`synchronize`关键字实质上是一种互斥锁：即任何时刻只有一个线程可以执行这段代码，它提供了如下几个保障:
- 可见性
- 互斥性
- 禁止重排序

JVM规范里对于实现互斥访问提供了`monitor`抽象，当一段代码进入临界区时它在某个对象上调用`monitorenter`,退出时调用`monitorexit`。  `monitor`同时还保证了拥有该`monitor`的线程可以多次重入(通过一个计数器实现)。而`monitor`是什么呢？

### `monitor`
#### 操作系统层次的管程
[`monitor`-wikipedia](https://zh.wikipedia.org/wiki/%E7%9B%A3%E8%A6%96%E5%99%A8_(%E7%A8%8B%E5%BA%8F%E5%90%8C%E6%AD%A5%E5%8C%96)):
> 管程 (英语：Monitors，也称为监视器) 是一种程序结构，结构内的多个子程序（对象或模块）形成的多个工作线程互斥访问共享资源。这些共享资源一般是硬件设备或一群变量。管程实现了在一个时间点，最多只有一个线程在执行管程的某个子程序。与那些通过修改数据结构实现互斥访问的并发程序设计相比，管程实现很大程度上简化了程序设计。 管程提供了一种机制，线程可以临时放弃互斥访问，等待某些条件得到满足后，重新获得执行权恢复它的互斥访问

由上面的介绍我们看到，监视器其实就是一种较为特殊的字程序，它通过某些手段保证了同一段代码同一个时间节点最多只有一个线程在执行。而在JVM中，则更进一步的增加了重入机制。 下面我们看下具体的实现：
#### `JVM`里`monitor`的实现
HotSpot的monitor实现类[`HotSpot JVM8 monitor`](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/runtime/objectMonitor.cpp)的基本结构如下：
```
  ObjectMonitor() {
    _header = NULL;
    _count = 0;
    _waiters = 0,
    _recursions = 0;  //重入次数
    _object = NULL;
    _owner = NULL;  //当前monitor的拥有者
    _WaitSet = NULL;  //处于Wait状态的等待队列
    _WaitSetLock = 0 ;
    _Responsible = NULL ;
    _succ = NULL ;
    _cxq = NULL ;
    _EntryList = NULL ;
    _SpinFreq = 0 ;
    _SpinClock = 0 ;
    OwnerIsThread = 0 ;
  }
```
线程通过CAS把`owner`字段由`null`变为非`null`表明拥有了monitor的所有权。当一个线程尝试进入监视器时，主要会经过如下的步骤:
-  通过CAS把当前线程设为monirotr的拥有者，成功则返回
```
cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
if (cur == NULL) {
 return ;
}
// static intptr_t cmpxchg_ptr(intptr_t exchange_value, volatile intptr_t* dest, intptr_t compare_value);
// static void* cmpxchg_ptr(void* exchange_value, volatile void* dest, void* compare_value);
//Windows下一种实现是:
inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {
  int mp = os::is_MP(); //查看是否是多核
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    //可以看到这个指令类似上面我们介绍的xchg指令
    cmpxchg dword ptr [edx], ecx
  }
}


```

- 如果`monitor`当前已经被锁定且拥有者是当前线程，则是**重入锁**,则累加重入次数:
```
if (cur == Self) {
    //这里有个整数溢出的bug
    _recursions ++ ;
    return ;
}
```

- 如果当前线程是第一次进入，则设置`_recursions`为1，`_owner`为当前线程
```
// 如果当前线程是第一次进入该monitor，设置_recursions为1，_owner为当前线程
if (Self->is_lock_owned ((address)cur)) {
    _recursions = 1 ;
    _owner = Self ;
    OwnerIsThread = 1 ;
    return ;
}
```

- 如果不是，那么则表明当前`monitor`被其他线程锁定，则通过自旋等待锁的释放
```
for (;;) {
  jt->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition()
  // or java_suspend_self()
  EnterI (THREAD) ;
  if (!ExitSuspendEquivalent(jt)) break ;
  _recursions = 0 ;
  _succ = NULL ;
  exit (false, Self) ;
  jt->java_suspend_self();
}
```
可以看到上面一个重要的方法是`EnterI`,这个方法的代码很长,核心其实就是要么自旋CAS要么`Park`，简要的逻辑如下
```

// 首先还是尝试获取锁一次。这个tryLock方法就是前面提到的对Atomic::cmpxchg_ptr的一次调用
if (TryLock (Self) > 0) {
    return ;
}


//调用一个带一定延时的，自旋的方式再次尝试获取锁（该优化可能需要对虚拟机实现非常了解才能知其所以然）
if (TrySpin (Self) > 0) {
    return ;
}
//把当前线程包装程ObjectWaiter，并尝试CAS插入到`_cxq`列表的前面.一但插入成功，除非获得锁否则
//始终在这个队列里
ObjectWaiter node(Self) ;
Self->_ParkEvent->reset() ;
node._prev = (ObjectWaiter *) 0xBAD ;
node.TState = ObjectWaiter::TS_CXQ ;
ObjectWaiter * nxt ;
for (;;) {
    node._next = nxt = _cxq ;
    if (Atomic::cmpxchg_ptr (&node, &_cxq, nxt) == nxt) break ;
    //入队成功，尝试获取下
    if (TryLock (Self) > 0) {
        return ;
    }
}
for (;;) {
   if ((SyncFlags & 2) && _Responsible == NULL) {
       Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
    }
    // park self
    if (_Responsible == Self || (SyncFlags & 1)) {
        TEVENT (Inflated enter - park TIMED) ;

        Self->_ParkEvent->park ((jlong) RecheckInterval) ;
        RecheckInterval *= 8 ;
        if (RecheckInterval > 1000) RecheckInterval = 1000 ;
    } else {
        TEVENT (Inflated enter - park UNTIMED) ;
 //由于上面的尝试获取锁还是失败了，睡眠当前线程等待被唤醒
        Self->_ParkEvent->park() ;
    }
    //到这里实际上是被唤醒了，再次尝试
    if (TryLock(Self) > 0) break ;
    //后面还有类似的自旋和fence之类的操作，不细述
}
```
到这里，可以看到`monitor`实际上是通过自旋或者`park`来保证对资源的互斥性访问。一个简单的流程图:
![](https://www.hollischuang.com/wp-content/uploads/2018/02/lockenter.png)
`monitorenter`指令申请当前栈上对象的锁，如果已经被当前线程获取则递增内部的一次计数器，否则等待。当内部计数器为0时则释放锁。而对于被锁住的块，Java编译器保证退出这部分代码块时会释放锁（即使说中间有未捕获的异常）
### `synchronized`的实现
上面我们说到`monitor`，这是JVM规范里定义的一种互斥性实现手段，而对于`synchronized`关键字，其实并不是直接调用监视器的，在JDK1.6之后对它的实现进行了一系列优化。具体如下：
1. 仅仅是同步代码块才会生成`monitorenter`和`monitorexit`指令。
对于同步方法，JVM不会生成任何特殊指令。当JVM调用一个方法时，它会自动判断方法是不是同步(`synchronized`)的，如果是，JVM在调用这个方法之前就申请获得锁。这里，被锁的对象就是当前的“作用域”，对于实例方法，就是当前实例对象；对于静态方法，就是对应的`Class`对象。当一个同步方法返回时，不管是正常还是异常，锁都会被释放。
2. 对于自动生成的获取锁代码，进行了几步优化已尽量避免进入性能影响相对较大的锁：
- 引入偏向锁
通过CAS设置对象头中JavaThread为当前线程id,成功则获取到锁。失败则等待到达全局安全点(safepoint)，撤销偏向锁，进入轻量级锁
- 轻量级锁
适用于多线程交替执行同步快，如果竞争激烈，则会进入重量级锁。
- 重量级锁
也就是上面提到的`monitor`的实现

这里具体的逻辑可以参考:
- [简书的一篇关于解释执行文章](https://www.jianshu.com/p/c5058b6fe8e5)
- [**hotspot官方的一篇文章**](https://wiki.openjdk.java.net/display/HotSpot/Synchronization), 这篇可以认真读下
- [hotspot的论文](https://pdfs.semanticscholar.org/1381/b7f3aebb605ebf00b7ff9158fae8d55b4e6e.pdf)
