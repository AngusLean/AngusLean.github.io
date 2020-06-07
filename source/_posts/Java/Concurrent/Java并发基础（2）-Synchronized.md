---
title: Java并发基础（2）-Synchronized
date: 2017/4/17
updated: 2018/1/24
tags:
   - 并发
categories:
   - Programming Language
   - Java
   - 并发
---
<!--more-->

##基础使用

基本上Java程序员都简单的了解synchronized的使用： 无非就是用在多线程环境下的同步。 看如下简单的例子：

```

public class 
UnsafeCounter {

    
private int 
count=0
;


    
public int 
getAndIncrement() {

        
return this
.
count
++;

    }


}
```

上面是一个简单的非常常见的POJO类，在多线程环境下的测试代码：
```

public class 
RunUnsafeCounter {


    
private static final 
UnsafeCounter 
unsafeCounter 
= 
new 
UnsafeCounter();


    
public static void 
unsafeCounter() 
throws 
InterruptedException {

        
int 
i=
0
;

        List<Thread> threadList = 
new 
ArrayList<>(
1025
);

        
while 
(i<
1000
){

            threadList.add(
new 
Thread(
new 
Runnable() {

                
@Override

                
public void 
run() {

                    System.
out
.println(Thread.
currentThread
().getName()+
" : "
+
unsafeCounter
.getAndIncrement());;

                }

            }));

            i++;

        }

        threadList.forEach(thread -> thread.start());

        
for
(Thread thread : threadList){

            thread.join();

        }

    }


    
public static void 
main(String[] args) {

        
for
(
int 
i=
0 
;i<
10 
;i++){

            
try 
{

                RunUnsafeCounter.
unsafeCounter
();
                
System.
out
.println(
unsafeCounter
.getCount());

            } 
catch 
(InterruptedException e) {

                e.printStackTrace();

            }

        }

    }

}
```
上面的测试类中有一个静态的```UnsafeCounter```实例，然后生成了1000个线程调用非线程安全的```getAndIncrement```方法 ， 按照平常单线程环境的结果，这里的值应该是1000. 但是运行```RunCounter```类就会发现结果不一定是 1000 并且每一次的结果都不一定相同。 这是因为多个线程同时访问 ```getAndIncrement```这一个非线程安全的方法，可能中间某几个线程可能同时在运行这个方法，然后在进行```++```操作时，某个线程获取到了当前值，结果又切换到了其他线程也获取到了当前值，然后这两个线程的 ```++```得到了相同的结果。 也就导致了最终结果的不确定性。
再看如下使用```synchronized```已保证线程安全性的代码：
```

public class 
SafeCounter {

    
private int 
count
=
0
;


    
public synchronized int 
getCount() {

        
return 
count
;

    }


    
public synchronized int 
getAndIncrement() {

        
return this
.
count
++;

    }


}
```
上面的POJO类的 ```getAndIncrement```方法使用 ```synchronized```修饰，而且```getCount```方法也使用 ```synchronized```修饰。 测试例子：
```

public class 
RunSafeCounter {


    
private static final 
SafeCounter 
safeCounter 
= 
new 
SafeCounter();


    
public static void 
safeCounter() 
throws 
InterruptedException {

        
int 
i=
0
;

        List<Thread> threadList = 
new 
ArrayList<>();

        
while 
(i<
1000
){

            threadList.add(
new 
Thread(
new 
Runnable() {

                
@Override

                
public void 
run() {

                    
safeCounter
.getAndIncrement();

                }

            }));

            i++;

        }

        threadList.forEach(thread -> thread.start());

        
for
(Thread thread : threadList){

            thread.join();

        }

    }


    
public static void 
main(String[] args) {

        
try 
{

            
for
(
int 
i=
0
;i<
10
;i++){

                RunSafeCounter.
safeCounter
();

                System.
out
.println(
safeCounter
.getCount());

            }

        } 
catch 
(InterruptedException e) {

            e.printStackTrace();

        }

    }

}
```


而上面的代码在经过10*1000次循环过后获得结果是10000， 无论重复多少次都是。 并且也保证了线程的安全性。（PS： 在看完下面的内容过后判断SafeCounter中的Getter方法的 ```getCount```方法如果去掉 ```synchronized```修饰会不会还是一样的结果？ ）


---


##规范说明
Java为多线程之间通信提供了非常多的机制，而其中， ```Synchronized```是最基础最简单的一个。在[JLS-17]( https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html)  对 ```Synchronized```的定义大概意思如下(ps为我加的备注，非原文)：


> synchoronized使用监视器实现。 Java中每一个对象都和一个监视器关联，线程可以锁或则解锁监视器， 同一时间只有一个线程持有监视器的锁，其他任何想获取该监视器锁的线程都会被阻塞知道可以获得该锁（ps: 拥有锁的线程释放过后）。 一个线程可能对一个监视器```锁```多次（ps: 可重入），每一个```解锁```恢复一次锁操作（ps: 内部维护一个监视器锁的次数，每退出一个减少1直到为0就释放该监视器的锁）
synchoronized块计算一个对象的引用，然后开始在对象的监视器上执行 ```锁``` 操作并且不继续向下执行直到 ```锁``` 操作成功后。然后 ， synchoronized块的内容开始执行。 如果块中的内容执行完成（不管是正常还是突然(ps: 被外部关闭之类)），在该监视器上就会自动执行 ```解锁```操作。
synchoronized方法在调用它的时候自动执行 ```锁```操作。它的方法内容在成功获取到锁之前不会执行，如果是实例方法，它锁住了**调用它的实例的监视器（方法中的this）**，如果是静态方法，它锁住了 ** 定义这个方法的类的Class对象的监视器 ** ，  如果块中的内容执行完成（不管是正常还是突然(ps: 被外部关闭之类)），在该监视器上就会自动执行 ```解锁```操作。
Java语言既不预防也不检查```死锁```（ps:这是程序员的事)
其他机制，比如```volatile```的读和写或则```java.util.concurrent```包提供了其他的可替代的同步方式。


[synchronized块]( https://docs.oracle.com/javase/specs/jls/se7/html/jls-14.html#jls-14.19 )：
> 一个synchronized块请求一个互斥锁，当拥有锁的线程在执行时，其他线程要获取这个锁必须等待。 它的语法如下：
    ```
        SynchronizedStatement:

            synchronized (Expression) Block

    ```

 表达式的类型必须为引用类型，否则编译期报错。该方法块 首先计算表达式的值，然后执行其中的代码。** 如果计算表达式突然结束，那么代码块已同样的理由突然结束。 如果是null，就会抛出空指针异常。**    否则，就获取到表达式值锁代表的对象的监视器的锁，然后开始执行同步代码块。 如果代码块正常退出，**监视器就会被解锁然后 synchronized块也正常退出。 如果是已其他任何理由突然中断的话，监视器会被解锁并且同步代码块会已同样的方式结束。**


[ synchronized方法 ]( https://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.4.3.6 ):
> 一个 synchronized方法在运行之前会先请求一个监视器（的锁）。对于一个静态方法，该类的Class对象关联的监视器将被获取。 对于一个实例方法， ```this```所代表的实例的监视器将被获取。


同样，在JLS中也写清楚了每一个对象关联的监视器都有一个```Wait Sets```，显而易见的就是用来保存当前等待获取当前监视器锁的线程集合。该集合仅仅可以被```Object.wait , Object.notify ,Object.notifyAll```操纵。


---
### synchoronized保证的互斥性与锁的对象
当然，对于 synchoronized来讲，它的具体的规范可以阅读一下，但也没有必要在这里完全照搬过来。 在理解了上篇的Java内存模型并且仔细阅读了上面的JLS中synchoronized的定义过后，对于在程序中如何正确的使用其实应该有了个基本的概念。 我认为，使用 synchoronized, 最基本也是最重要的就是:
- 你为什么需要用 synchoronized？
- 你```锁```的究竟是哪个对象？
- 为了做什么？


考虑如下代码：
```

public class 
SynchronizedCounter {

    
private int 
c 
= 
0
;


    
public synchronized void 
increment1() {

        
c
++;

    }

    

    
public void 
increment2() {

        
synchronized
(
this
){

            
c
++;

        }

    }

}
```

对于 increment1方法，它是一个同步方法，并且是实例方法。 根据上面的定义，该方法会获取调用该方法的实例的监视器的锁； 而对于 increment2，它是一个同步代码块，但获取一个对象的引用，然后尝试获取锁。 这里的引用是this,其实也就是该调用 increment2的实例。  所以说，  increment1和 increment2其实是做了完全一样的事情。
代码:
```

class Test {
    int count;
    synchronized void bump() {
        count++;
    }
    static int classCount;
    static synchronized void classBump() {
        classCount++;
    }
}
```

与
```
class BumpTest {
    int count;
    void bump() {
        synchronized (this) { count++; }
    }
    static int classCount;
    static void classBump() {
        try {
            synchronized (Class.forName("BumpTest")) {
                classCount++;
            }
        } catch (ClassNotFoundException e) {}
    }
}
```
也是拥有相同的效果。
在搞清楚锁的对象和时间周期过后，下面代码的安全性应该很容易看出来了：

```

public class 
LockObjectTest {

    
private static int 
index
=
0
;


    
public synchronized int 
getAndIncrement1(){ //这个锁的是实例的监视器

        
return 
index
++;

    }


    
public static synchronized int 
getAndIncrement2(){ //这个锁的是LockObjectTest类的Class对象
的监视器

        
return 
index
++;

    }


    
public static void 
main(String[] args) {

        
int 
i=
0
;

        List<Thread> threadList = 
new 
ArrayList<>(
1000
);

        LockObjectTest lockObjectTest = 
new 
LockObjectTest();

        
while 
(i<
10000
){

            i++;

            threadList.add(
new 
Thread(
new 
Runnable() {

                
@Override

                
public void 
run() {

                    
lockObjectTest
.getAndIncrement1();

                }

            }));

            threadList.add(
new 
Thread(
new 
Runnable() {

                
@Override

                
public void 
run() {

                    LockObjectTest.
getAndIncrement2
();

                }

            }));

        }

        threadList.forEach(thread -> thread.start());


        
try 
{

            
for
(Thread thread : threadList){

                thread.join();

            }

        } 
catch 
(InterruptedException e) {

            e.printStackTrace();

        }


        System.
out
.println(LockObjectTest.
index
);

    }

}
```
### synchoronized保证的内存可见性

> 当线程A执行一个同步代码块过后，线程B进入同一个监视器锁的同步代码快的时候，所在线程A的操作（特别是对变量的改变）都保证可以被线程B看到（即不会因为重排序或则缓存之类的影响而看到错误的值）


内存可见性在单线程环境下从来没有出现过，因为这似乎就是一个智障问题：我前面给变量赋值了，后面肯定可以看到这个值。不然我们的代码岂不是问题重重？
而在多线程环境下之所以会出现这个问题还是由于编译器、运行时、CPU共同作用的结果。
- 编译器（不一定指javac，JIT）会对代码进行优化，一个非常常见的就是[编译器循环优化，知乎 RednaxelaFX 的一个回答 ]( https://www.zhihu.com/question/39458585/answer/81521474 )。 编译器在编译的时候可能就已经改变了代码中的变量声明或则赋值顺序-只要保证了语义一致性。 R大已经解释的非常清楚。
- 现代处理器的[乱序执行]( https://en.wikipedia.org/wiki/Out-of-order_execution )和CPU上越来越多的缓存（L1,L2,L3 cache）都使得你最终跑在CPU上的代码和你所写的出入较大。 多线程环境下尤其需要考虑这种影响。 比如下面的代码：


```
int arith ( int x , int y , int z ) {  
  int t1 = x + y ;
  int t2 = z * 48 ;
  int t3 = t1 & 0xFFFF ;
  int t4 = t2 * t3 ;
  return t4 ;
}

```
由于t1和t2的赋值互不影响，所以他们的顺序完全可能已随机的次序跑在CPU上。
而内存可见性其实也是这个道理。 当你的程序跑在同一个线程的时候，后面的代码读取之前对变量的更改都会是在同一个“核心”的寄存器或则缓存上。 而如果是多线程环境，假设某一个线程更改了某个变量，然后放到了它的寄存器上。 而另外一个线程此时来读取这个变量，它是会从内存中读取还是从这个 “核心” 的缓存中读取还是从这个 “核心”的寄存器上读取、又或则由于重排序这里的赋值还没有发生 是不能得到保证的。而唯一可以确定的是，它读取到的总会是某个线程在某个时间更改的数据，这被称为最低保证性(JMM规定了64位的数值（double,long）可以分成2个32位的进行计算，也就是说这两种数据类型没有最低保证性。它们的数据完全可能是随机的)。
如下代码：

```

public class 
NoVisibility {

    
private static boolean 
ready
;

    
private static int 
number
;

    
private static class 
ReaderThread 
extends 
Thread {

        
public void 
run() {

            
while 
(!
ready
)

                Thread.
yield
();

            System.
out
.println(
number
);

        }

    }

    
public static void 
main(String[] args) {

        
new 
ReaderThread().start();

        
number 
= 
42
;

        
ready 
= 
true
;

    }

}
```
上面代码主线程和读线程访问共享变量```ready```和```number```，主线程开始读线程，然后把```number```设为42，把```ready```设置为```true```。 读线程等待ready为true后打印```number```. 但是这里，读线程可能会看到```number```是42也可能是0，或者说是永远不终止。主线程对于ready和number的写不能保证可以被其他线程看到。 
***synchronized可以保证内存可见性 *** ，也就是使用了 synchronized关键字的方法或则语句都可以保证内存可见性（还有其他机制，如```volatile```）。具体的细节 并发编程网有一篇非常好的[文章]( http://ifeve.com/easy-happens-before/#more-8223)
当线程A运行一个 ```synchronized```块，然后之后线程B进入同一个锁的 ```synchronized```块时，线程A释放锁之前可见的变量可以保证在线程B获取锁的时候可以看见。 换句话说，线程A做的事情线程B都知道。 而没有 ```synchronized```，则没有这样的保证。


 