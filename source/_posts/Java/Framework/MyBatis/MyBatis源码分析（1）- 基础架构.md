---
title: MyBatis源码分析（1）- 基础架构
date: 2017/11/23
updated: 2017/12/7
tags:
   - javaee
   - mybatis
categories:
   - Programming Language
   - Java
   - Framework
   - MyBatis
---
作为最为流行的ORM框架之一，Mybatis一直占据着半壁江山。 得益于Mybatis极其灵活的SQL书写规则和很方便的接口方式，Mybatis在灵活性与功能、易用性上取得了较好的平衡。 **知其所以然**是每个程序员都应该努力掌握的技巧，本系列文章将逐步分析Mybatis整个源码包， 理清这个“短小精悍”的库的实现原理。

本节聚焦于Mybatis(3.4.5版本)整体架构，将理清Mybatis主要有哪些模块，主要的架构。 <!--more-->


## 1.架构



![架构图](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-source-analyze-basic.jpg)

上图为逻辑上的Mybatis架构（个人意见，欢迎讨论）， 当然， 对照着这张图和源码，几乎可以一眼看出那些类属于哪个模块。
以下分别说明不同模块的功能：
### 1.1 接口层
根据官方文档，Mybatis的使用方式：
```java
String resource = "org/mybatis/example/mybatis-config.xml";

InputStream inputStream = Resources.getResourceAsStream(resource);

SqlSessionFactory sqlSessionFactory =

new SqlSessionFactoryBuilder().build(inputStream);

```
其中，` SqlSessionFactoryBuilder`这个builder事实上就是构建了一个`Configuration`,而这个类将伴随Mybatis上下层，这个类后面再介绍。这里的` SqlSessionFactory  `就是用来构建我们的SQL执行类`   SqlSession`的一个工厂类。 
`   SqlSession`提供就是我们使用Mybatis时最主要接触的类，他提供了通过mapper的增删改查的功能。


### 1.2 数据处理层

上面说道` SqlSession`提供了通过mapper的增删改查，但是他自身的实现并不是直接处理任何实现逻辑。比如我们在源码中随机选取查询数据的接口的实现。</br>


```java

@Override

public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {

    try {

      MappedStatement ms = configuration.getMappedStatement(statement);

      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);

    } catch (Exception e) {

      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);

    } finally {

      ErrorContext.instance().reset();

    }

  }

```

可以看到有3个很明显的任务划分:`Configuration`,`MappedStatement`，`Executor`。
 简要的说，如下任务都是“外包”给其他接口处理：
1. 配置文件解析
2. mapper文件解析
3. mapper接口与mapper xml映射
4. 数据库操作时参数解析映射到生成最终SQL
5. 数据库操作（JDBC之类）

上面其实就是一个ORM框架最主要的流程。 对应到Mybatis，就是：
- 各类文件解析与mapper接口的映射都是在 ```Configuration```及其构建者中实现的，这也是为什么这个类在架构图中占据着2/3的纵向位置。
- 请求参数相关处理在```MappedStatement```类
- SQL执行在```Executor```接口。 

后面将分别讲述上述接口的实现。

### 1.3 核心层
Mybatis的核心层主要有如下一些包：
- 日志。  为了适配log4j, commons logging, slf4j等， Mybatis内部实现了一个日志模块。
- IO。 比如对于配置文件， xml资源文件的读写。
- reflection。 最典型的就是调用接口实际上是执行对应的mapper statement，很明显需要通过反射来实现。
- exception。 这个也是框架中必须处理的一个东西。SQL执行本身有很多异常，而作为一个ORM框架肯定不能让这些异常直接抛到上层，一般都是转化为自己定义的```RuntimeException```。
- 缓存。 Mybatis提供了两种缓存：
    -  ```   SqlSession```级别。 对于同一个 ```   SqlSession```，相同的查询不会反复查询数据库而直接使用缓存中的数据。 这个缓存就是一个简单的Map,通过```MapperStatement```的配置参数来决定是否清楚缓存。 该缓存默认开启，并且建议不要手动关闭。

    - 二级缓存。 该缓存必须手动开启， 在mapper xml中添加```<cache/>```。 该缓存类型为LRU(least recently used)，实现较复杂，后面会将。

- 数据源。 出于如下考虑：
    - 每次操作数据库不可能都新建一个连接， 自身最好可以维护一个连接池。

    - 适配现有连接池。 比如用的很多的```druid```。

- 注解。 Mybatis支持直接在接口上面注解相应的SQL， 自然需要相应的支持。 不过在项目中不建议这么使用， 不利于维护。
- 事物。 Mybatis自身定义了几种事物级别， 不支持隔离事物这种操作。 不过在实际使用中，大部分都是通过Spring来管理。

## 2 其他
根据上面对架构上的逻辑梳理，所以我觉得在Mybatis中最重要的类有如下几个：
- `Configuration`
- `SqlSession`
- `Executor`

其他的类都可以理解为为了达成上述3个类的目标而建立的工具， 比如各种工厂类，建造类，代理类什么的。 下篇将首先讲述在通过一个mapper接口操作数据库时的流程，已解开个人觉得最神秘的这层面纱。
同时， 鉴于`Spring`在java中基石一样的地位， `Mybatis`毫无疑问有个`mybatis-spring`的东西存在。 后面将专门讲述下这个包装模块做的事情，不过相较于mybatis本身，在理解了mybatis的源码过后，mybatis-spring做的事情也可以说是一目了然了。

---
最后废话下为什么要“分析源码”， 我的个人观点是很多人在项目开发往往是没有机会接触到框架/库的开发的，并且业务逻辑往往也很简单，一个典型的例子就是单例工厂是仅有的设计模式，Dao/Service/Controller分层是所有的架构。 当然，这样的代码或者说架构肯定有其存在的意义，但是，对于开发者个人来讲是极其不利的。 没有足够复杂的业务量，足够大的架子来写，开发者本身学习过的很多东西都会逐渐忘却，但是开发能力却完全没有随着项目经验的增加。没有足够的分析能力、抽象能力、架构能力，毫无疑问也是没有多少竞争能力。</br>
而源码学习是解决这些问题的一个可能的最方便的途径， 通过很多库或者框架的源码，能学习到的东西着实很多。 往大了说，可以看到其他优秀的coder是如何架构一个复杂的项目，如何抽象整个系统，如何组织代码结构，如何控制代码质量等等；往小了说，如何设计类层级结构，如何使用设计模式来代码上的各种问题，乃至于方法、变量命名等等。如果看的更加仔细的，还会看到很多不设计库/框架不会设计到的问题: 比如java bridge。还有很多很多的东西。 套用《深入理解MFC》作者侯杰先生的话： **源码面前，了无秘密**。还有一个有意思的[讨论](https://www.zhihu.com/question/24997123)
当然，很多库/框架往往需要处理很多细节上的东西，阅读源码的时候很容易就掉入了繁冗的细节中。这时候需要带着强烈的**目的**去看才行，要学会绕过不相干的东西，带着思考阅读才是正确的方式。
最最后的废话，阅读源码需要你至少使用过这个东西才行。Mybatis作为一个称的上“小而精”的库，又被广泛使用，从它入手会是一个比较容易的过程。


---

问题：
- 看到上面的图怎么和源码结构联系起来？    
- `SqlSession`实例的获取为什么需要3步？  一直保持一个`SqlSession`的实例是否合适？ 使用的设计模式？
- 配置文件类为什么需要是全局的？ 