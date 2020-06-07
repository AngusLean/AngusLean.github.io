---
title: MyBatis源码分析（4）- SqlSession接口增删改查的实现原理(2)
date: 2017/12/3
updated: 2017/12/20
tags:
   - javaee
   - mybatis
categories:
   - Programming Language
   - Java
   - Framework
   - MyBatis
---
上篇分析到了`SimpleExecutor`接口， 这个接口中的`doUpdate,doQuery`方法是`Executor`体系中最终的实现。 本节分析下`Executor`中具体的逻辑

<!--more-->

## 1. `SimpleExecutor`接口
### 1.1 `doUpdate, doQuery`的实现

```java
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }

  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }
```
上面的3行代码就是由`BaseExecutor`类中的`query,update`调用到`SimpleExecutor`的方法。可以看到对于`doUpdate,doQuery`两个方法都有如下几步：
- 从`Configuration`类中生成一个对应的`StatementHandler`
- 获取一个JDBC连接，然后调用`handler`的`prepare`方法，该方法事实上是调用的`connection.createStatement`创建的连接。handler接口下面讲述
- 然后调用`handler`的对应`query，update`方法
- 再返回数据给`BaseExecutor`中的`query, update`方法当中

显而易见，`StatementHandler`接口实际承担了`Statement`的生成以及`query, update`的操作。 

## 2. `StatementHandler`接口
### 2.1 整体结构

![](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-source-analyze-statement.png)

### 2.2 调用分析

#### 2.2.1 介绍

`StatementHandler`接口的继承链与`Executor`接口的非常相似，都是包含一个委托类，一个抽象基类以及3哥实现类。调用方式有些许不同：
- 对`StatementHandler`的`query, update`的调用最终会传到`SimpleStatementHandler, PrepareStementHandler, CallableStatementHandler`中的一个
- `RoutingStatementHandler`的构造函数起到了工厂的作用，它根据`statement`的类型决定生成`SimpleStatementHandler, PrepareStementHandler, CallableStatementHandler`哪一个实例。
下面分别看下这3个类的实现方式

#### 2.2.2 `SimpleStatementHandler`

```java
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    return resultSetHandler.<E>handleResultSets(statement);
  }
  public int update(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    int rows;
    if (keyGenerator instanceof Jdbc3KeyGenerator) {
      statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else if (keyGenerator instanceof SelectKeyGenerator) {
      statement.execute(sql);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else {
      statement.execute(sql);
      rows = statement.getUpdateCount();
    }
    return rows;
  }
```

#### 2.2.3 `PrepareStementHandler`

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.<E> handleResultSets(ps);
  }
public int update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    int rows = ps.getUpdateCount();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
  }
```
#### 2.2.4 `CallableStementHandler`
```java
public int update(Statement statement) throws SQLException {
    CallableStatement cs = (CallableStatement) statement;
    cs.execute();
    int rows = cs.getUpdateCount();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    keyGenerator.processAfter(executor, mappedStatement, cs, parameterObject);
    resultSetHandler.handleOutputParameters(cs);
    return rows;
  }
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    CallableStatement cs = (CallableStatement) statement;
    cs.execute();
    List<E> resultList = resultSetHandler.<E>handleResultSets(cs);
    resultSetHandler.handleOutputParameters(cs);
    return resultList;
  }
```
#### 2.2.5 总结
上面3个类结合JDBC的API一看就知道是怎么回事，这3个类就是对JDBC的SQL执行的一层封装。
所以到目前为止，我们看到了`SqlSession`接口调用mapper xml的执行链：*sqlsession -> executor -> statementhandler*,最终会根据具体SQL的类型调用JDBC的三种statement类型来处理实际业务。 `MappedStatement `封装了mapper xml中声明的每一个SQL。 `StatementHandler`封装了JDBC操作。 `Executor`封装了mapper xml缓存以及执行，批量执行。
合理的策略模式、抽象基类 非常完美的阐释了java中的OO思维。 

