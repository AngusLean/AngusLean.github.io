---
title: Mybatis源码分析（8）-mybatis-spring的事务管理
date: 2018/1/18
updated: 2018/1/29
tags:
   - javaee
   - mybatis
   - mybatis-spring
categories:
   - Programming Language
   - Java
   - Framework
   - MyBatis
---
在上面一节[MyBatis源码分析（7）]中看到，我们一般会使用mybatis的`SqlSessionTemplate`来作为mapper的生成类，该类实际上就是`SqlSessionFactory`的一个代理类。在官方文档中，也明确写明了该类是mybatis-spring的核心类。
在上一节中，我们可以看到该类的`insert,select, getMapper`等方法。 本节看下具体代理实现。
该类内部通过`SqlSessionInterceptor`类来代理`SqlSessionFactory`。

<!--more-->

我们看到这个类的`invoke`方法是这样的：
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
//1. 创建一个SqlSession实例
    SqlSession sqlSession = getSqlSession(
        SqlSessionTemplate.this.sqlSessionFactory,
        SqlSessionTemplate.this.executorType,
        SqlSessionTemplate.this.exceptionTranslator);
    try {
    //2. 调用具体方法
        Object result = method.invoke(sqlSession, args);
    //3. 判断是否需要提交
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
            sqlSession.commit(true);
        }
        return result;
    } catch (Throwable t) {
    //4. 包装异常
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
            closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            sqlSession = null;
            Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
            if (translated != null) {
                unwrapped = translated;
            }
        }
        throw unwrapped;
    } finally {
    //5. 关闭session
        if (sqlSession != null) {
            closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
    }
}
```
上面的代码是任意的一个Mapper的方法调用或者说通过`SqlSessionTemplate`调用任何方法时候的代理方法，可以看到如下几个步骤（也就是mybatis-spring的核心）：

1. 获取一个`SqlSession`实例
2. 调用具体方法
3. 如果当前没有处于spring事务管理，则直接提交（否则由spring管理这个事务）
4. 关闭session
5. 包装异常为spring的异常

可以看到，这几步其实也是mybatis-spring的核心功能。

### mybatis的事务如何嵌入到spring事务管理

这个关键就在上面的第1步 - 获取一个`SqlSession`实例。 我们看下实现：

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    //1.从spring事务管理容器中获取一个Holder
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
   //2.获取session
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
        return session;
    }
   //3.如果从事务管理器中没有获取到，就重新创建一个
    session = sessionFactory.openSession(executorType);
   //4. 注册到holder
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
}
```

对spring的事务管理熟悉点的很容易看出来，这个逻辑与spring的`DataSourceUtil`异曲同工： 使用一个holder类包装具体类型（此处`SqlSession`），然后通过spring的同步事务管理器(TransactionSynchronizationManager)获取这个holder， 从holder里面获取需要的对象，如果同步事务管理器没有获取到则新建一个并且绑定到spring的同步事务管理器中。

这里介绍下`TransactionSynchronizationManager`。 该类是线程级别的资源和事务的同步管理器，通常被资源管理器使用。他的典型使用场景是某些线程独立的、可复用的资源管理。可能说的不是很清楚，官方文档里这样描述：
>central helper that manages resources and transaction synchronizations per thread 。。Resource management code should only register synchronizations when this manager is active, which can be checked via isSynchronizationActive(); it should perform immediate resource cleanup else. If transaction synchronization isn't active, there is either no current transaction, or the transaction manager doesn't support transaction synchronization.

具体到上面的例子来,`TransactionSynchronizationManager`对于同一个`sessionFory`，在事务激活的情况下，同一个线程总是返回相同的`SessionHolder`。
比如对于一个**开启了事务**的service的方法，这个方法可能调用多个Mapper接口，那么它们会在同一个事务管理中，`SqlSessionHolder`类则维护了一个`SqlSession`的引用次数，它辅助了`TransactionSynchronizationManager`会处理所有数据库连接的获取与释放，以及回滚，并且保证线程安全。这样就可以避免连接泄漏。
`SqlSessionHolder`类则维护了一个`SqlSession`的引用次数，它主要起到一个辅助作用， 以便于在一个事务中复用资源-也就是`SqlSession`。

### 如何开启mybatis-spring的自动事务管理
既然是使用的spring的事务管理，首先在配置中开启。通常的做法是在配置文件中加入如下的配置：
```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSource" />
</bean>
```
这样就开启了spring的事务管理。 然后，使用`@Transactional`注解或者`AOP`方式标注具体的事务处理位置即可。

而对于mybatis-spring, 则会自动的使用spring的事务管理。注解或者`AOP`标明的位置中的mybatis操作，都会被spring事务管理所管理。文档在[这里](http://www.mybatis.org/spring/transactions.html)

必须注意的是，mybatis-spring的连接在你的项目中没有开启spring事务的情况下，依然存在着连接泄漏的情况。 上面一直有提到，**使用`@Transactional`注解或者`AOP`方式标注具体的事务处理**这个必须要设置才行。