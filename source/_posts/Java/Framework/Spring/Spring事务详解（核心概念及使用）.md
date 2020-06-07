---
title: Spring事务详解（核心概念及使用）
date: 2018/5/25
updated: 2018/5/31
tags:
categories:
   - Programming Language
   - Java
   - Framework
   - spring
---
# spring事务
## 基本知识
在数据库操作中，经常需要某些连续性的读和写，并且他们之间还有一定的先后关系，逻辑上来说，对于应用程序，他们是同一个“单元”。但是，每次数据库的CRUD都是可能出错的，比如网络问题、服务器宕机、资源泄漏等等， 可能说对于某单个的查询我们可以专门编写相应的处理逻辑，但是对于**“一系列”**的操作，这个工作量无疑是颇有难度和工作量的。 这就是事务要解决的问题：


<!--more-->



>A transaction is a way for an application to group several reads and writes together into a logical unit. Conceptually, all the reads and writes in a transaction are
executed as one operation: either the entire transaction succeeds (commit) or it fails (abort, rollback). If it fails, the application can safely retry
事务是应用程序把一系列的读和写组成**一个**逻辑单元的一种手段。或者说，事务使得其中的所有的读和写被当作**一个操作**来执行，要么全部成功(`commit`)， 要么全部失败（`abort, rollback`），如果失败了，应用程序也可以优雅的处理后续逻辑。


上面是`Designing Data-Intensive Application`的书中关于事务的解释，足够简明清晰。并且开发中，肯定也都或多或少的使用过事务，不多赘述概念。
Java世界中，事务相关的方案以及不少了： JDBC transaction, JPA, Hibernate等等。而作为Java企业级开发重量级的组件-Spring， 为什么还要做个Spring Transaction呢？ 它解决了哪些问题，是如何解决，以及如何使用呢？

### 解决的问题
上面说过，Java中事务单就广泛使用的，就包括了JDBC，JPA, Hibernate。 他们每一种都有不同的API,不同的使用方式，不同的容器要求等等。但是事务的初衷-- 它不是天然就存在的，就是为了简化应用程序的编程模型而设计的。 所以，对于应用程序友好是一个非常重要的标准。
作为应用开发的一个很通用的需求-事务管理，在Java中有这么多不同的解决方案其实并不方便，何况说，绝大多数情况下，绝大多数人的事务管理需求都是一致的。  诸如分布式事务、多数据源这些较为复杂的需求并不多， JPA等方案相对来讲复杂性过高，而复杂性带来的feature却不一定是有用的。 所以，有了Spring Transaction。

### 设计目标
Spring框架为事务管理提供了一个持久层抽象，它有如下的优点:
- 横跨JTA, JDBC, Hibernate, JPA, JDO的统一的持久层 抽象
- 支持声明式事务
- 比JPA等易于使用的API接口
- 与Spring Data Access抽象无缝衔接

### 设计思路


## 使用方式
### 声明式事务
#### 基础概念

#### `Propagation`事务传播级别
`Propagation`,事务传播级别。 它指的方法调用过程中，事务传播的规则。比如说在事务标记的方法`ServiceA.Method1`中调用另外一个事务标记的方法`ServiceB.Method1`时的事务行为： 是就用当前这个事务呢，还是新建一个等等。 Spring对此提供了良好的支持。事务传播级别共分为7种,其中最为重要的前面两个，着重介绍：

- `REQUIRED` 
![REQUIRED级别事务传播](http://p5rx80fa6.bkt.clouddn.com/tx_prop_required.png)
`REQUIRED`级别会当前调用链中所有的事务标记的方法使用一个物理事务（比如JDBC事务），它的策略是：
 - 如果当前调用栈没有事务，就会新建一个事务
 - 如果当前调用栈已经有事务，那么则会主动参与到这个事务中去
这个级别是默认的级别，也是最常用的。比如说，在某个`Service`层方法中可能会调用多个`Dao`层的方法，那么在这个级别的时候，对于这一个`Service`方法所调用的所有`Dao`方法，都是处于一个事务中。 也就是任何一个被调用的方法出现在`RunTimeException`，都会回滚整个方法，也就是这个事务的范围

> 默认情况下，一个事务会使用外层事务的配置。也就是说,`isolation level, timeout, read-only`这些配置都是使用外层的配置。

当传播级别设置为`REQUIRED`，对于每一个被标记这个注解的方法来说都会创建一个逻辑上的事务范围，其中每一个逻辑上的事务范围都可以独立决定是否`RollBack Only`,外层事务与内层的事务互相独立。当然，这些“逻辑上”的事务其实都对应着同一个物理事务（比如JDBC事务）。也就是说，即使内部某个事务标记为`RollBack Only`，外层事务依然可以决定执行`Commit`而不是`Roll Back`。所以说如果内部事务因为任何原因设置了`RollBack Only`，外部事务依然提交的话就需要收到一个异常:`UnexpectedRollbackException`来标明内部发生了错误，同时，这也是数据库事务`ACID`中的A，`atomicity`原子性的要求。
Oracle文档中关于标记为`RollBack Only`的[说明](https://docs.oracle.com/cd/E26180_01/Platform.94/ATGProgGuide/html/s1204markingrollbackonly01.html):
> 当发生`RunTimeException, Error`时，应用程序可以决定当前事务是否需要被回滚。 但是，应用程序不应该直接回滚当前事务，相反，它应该标记当前事务为`RollBack Only`，也是就是设置一个标识表示当前事务不应该被提交。
当事务走到了最后的时候，应用程序检查事务是否设置了`RollBack Only`，如果没有设置，则提交事务(`commit`)，成功或者导致回滚。如果发现设置了这个标志，之后的任何资源操作的动作应该导致错误，检查这个标志可以减少某些错误和调试的时间。

如果希望内层事务仅仅捕获某些异常，可以使用`Transactional`的`rollbackfor,noRollBackFor`配置。或者说配置事务传播级别为`REQUIRES_NEW`来新创建一个事务。

- `REQUIRES_NEW`
![require new transaction](http://p5rx80fa6.bkt.clouddn.com/tx_prop_requires_new.png)

`PROPAGATION_REQUIRES_NEW`,与`PROPAGATION_REQUIRED`最大的不同是每个事务范围都会创建一个**新的物理事务**。不会参与已经存在的外层事务。所以内层事务可以独立的提交、回滚，而与外层事务互不干扰。内层事务完成过后将立即释放锁（因为当外层事务进入内层事务的时候，会创建一个新的事务，当前事务则被挂起）

- `NESTED`
嵌套事务，与`PROPAGATION_REQUIRES_NEW`的不同是它与外部事务使用的是同一个物理事务，但是在事务的入口会创建`SavePoint`,也就是说内层事务会创建单独的还原点，每个内层事务可以回滚到进入事务的时候。而外层事务可以在内层事务以及发生回滚的情况下继续执行。这个级别其实是JDBC的一个功能，所以也仅仅在使用`DataSourceTransactionManager`这个事务实现时有效。
- `MANDATORY`
当前方法必须运行在一个事务中，如果没有则抛出异常。
- `NEVER`
当前方法不能运行在一个事务中，如果由则抛出异常。
- `NOT_SUPPORTED`
不支持
- `SUPPORTS`
当前方法不需要事务，但是如果有一个事务存在，它也可以运行。

必须注意的是，使用`Transactional`注解的方法的事务**仅仅在外部调用的时候生效**，也就是类似这样的代码不会让事务生效：
```java
public class TestService {
    public void testA(){
        //调用类里面的另外一个事务方法。 实际上事务不会生效
        testB();
    }

    @Transactional
    public void testB(){
        System.out.println("TEST B");
    }
}
```

这里无论方法`testB`使用的事务传播级别是什么，都不会生效。仅仅在通过`TestService`的实例调用`testB`方法时事务才会生效。同时，对于同一个类的不同方法，事务会已外层为准，而不会根据内部方法的事务级别更改：

```java
public class TestService {
    @Transactional
    public String test3() {
        System.out.println("test1");
        test2();
        return "test3";
    }
    //即使事务传播级别是新建事务，但是由test3方法调用自身时会使用test3的事务，不会新建事务。
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public String test2() {
        System.out.println("test2");
        return "test2";
    }
}
```

而如果是调用其他类的方法，则传播级别会生效:
```java
public class TestService {
    @Transactional
    public String test1() {
        System.out.println("-----------------------");
        testService.test();
        System.out.println("test1");
        return "test1";
    }
}

@Component
public class TestService {
        //会创建一个新事务
        @Transactional(propagation = Propagation.REQUIRES_NEW)
        public void test(){
            System.out.println("==============TestService - test");
        }
}
```

#### `Isolation` 事务隔离级别
通常情况下，隔离级别与事务传播级别互相搭配使用。传播级别配置了事务之间的创建关系，隔离级别配置事务之间的“可见性”关系。
- `DEFAULT`， 默认级别，该级别会使用底层的数据存储结构的隔离级别。对于JDBC,默认是`READ_UNCOMMITED`
- `READ_UNCOMMITED` 可以脏读，也就是一个事务可以读取到另外一个事务中没有提交的数据；`non-repeatable read, phantom read`都可能发生
- `READ_COMMITED` 不能脏读，一个事务仅能看到其他事务已经提交的数据; 但是`non-repeatable read, phantom read`可能发生。也就是事务中两次读取某一行内容可能不一致（中间其他事务更改了数据）；可以防止 `dirty write`，通常通过延迟第二次写到第一次写的事务已经提交或者中止的时候 。
- `REPEATABLE_READ`  可以防止脏读，`non-repeatable read`。
- `SERIALIZABLE` 可以防止脏读，`non-repeatable read, phantom read`。
> `non-repeatable read`,发生在当一个事务A从数据库从查询到某行数据(row)， 然后事务B随后就更改了这行的数据。 事务A再次读取改行数据时，就可能读取到与第一次不同的数据。
`phantom read`，也译作幻读。 事务A从数据库中查询到获取某种条件的几行数据(multi row)，事务B随后插入或者更改了满足了事务A查询条件的数据，事务A再次以同样的条件查询数据时，就可能看到多余的行。这个行被称为“幽灵(phantom )行”
`dirty write`, 脏写。当两个事务并发写某个对象时，我们不会知道他们写的顺序，但是可以知道的是后面一个写会覆盖前面一个写。但是如果第一次写是在一个未完成的事务里，当另外一个事务对相同对象发出写的时候，第二次写如果覆盖的是一个“未提交(`uncommited`)"的数据就称为脏写。 因为这种情况会导致并发写的时候的数据错误。[参考](https://en.wikipedia.org/wiki/Write%E2%80%93write_conflict)

上面几个事务隔离级别从上往下以此变得更加严格，同时，对系统资源的占用也逐渐增加，也增加了一个事务阻塞另一个事务的可能性。它也是数据库事务四大特性的一个（ACID）。
### 编程式事务






### 参考
[spring文档](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/transaction.html)
[oracle文档](https://docs.oracle.com/javase/tutorial/jdbc/basics/transactions.html)
[博客](http://elliot.land/post/sql-transaction-isolation-levels-explained)