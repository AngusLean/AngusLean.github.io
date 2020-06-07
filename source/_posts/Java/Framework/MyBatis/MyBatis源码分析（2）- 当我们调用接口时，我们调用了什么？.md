---
title: MyBatis源码分析（2）- 当我们调用接口时，我们调用了什么？
date: 2017/11/27
updated: 2018/1/10
tags:
   - javaee
   - mybatis
categories:
   - Programming Language
   - Java
   - Framework
   - MyBatis
---
通过mapper interface使用Mybatis应该是最为通用的一种使用方式，通过这种方式阅读起来清晰明了，同时又合适的屏蔽了细节。 但是，Mybatis是如何让自定义接口可以实现种种数据库操作的功能，是一个非常有趣的问题。


<!--more-->


## 1. 开始

但是这节我们不直接从接口开始，先从架构图中最“大”的一部分-配置 开始说起。


作为Mybatis中**牵连**最广的一个类，`Configuration`类承担起了如下3个任务：

- 为所有类提供可自定义的配置的读取的功能
- 读取相关配置文件：
    - mybatis配置文件，比如mapper文件位置，插件等等
    - mapper xml文件的读取
- mapper interface与xml的映射

之所以把配置这个类放在最前面而不是`SqlSessionFactory`,是因为这个这个工厂类的建立者`SqlSessionFactoryBuilder`事实上就是调用 `XMLConfigBuilder`类构建了一个`Configuration`。
而Mybatis使用mapper interface的方式很简单


    SqlSession session = sqlSessionFactory.openSession();
    try {
      BlogMapper mapper = session.getMapper(BlogMapper.class);
      Blog blog = mapper.selectBlog(101);
    } finally {
      session.close();
    }



所以我们这里关注mybatis为何能获取到这个接口的实例，获取到的实例到底是什么类型以及是如何影响到数据库的。


## 2. 整体结构
前面说过， `Configuration`类维护了mapper xml与mapper接口的映射关系， 但是事实上 `Configuration`类的成员变量显示并没有类似`Map<String, Class>`的成员，那么他们是怎么关联起来的呢？
同时，当我们通过`SqlSession`获取到一个mapper的接口实例时， 对这个接口的方法调用是如何映射到对应的mapper文件中的xml声明呢？
这里我们先跳过上面所列出的配置文件与mapper xml文件读取， 先看下对一个接口中某个方法调用一系列的流程：


![](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-source-analyze-mapper.png )


## 3. 主要流程：

1.`Configuration`类持有了一个包含所有mapper xml的映射结果集， 一个`MapperRegistry`实例。

2.`MapperRegistry`实例持有所有的mapper接口的`Class`与`MapperProxyFactory`映射结果集
这里之所以是一个工厂而不是直接生成一个代理类，是因为每个代理类都需要与`SqlSession`关联，每次调用都不同，所以需要动态生成

    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
      }

3.`MapperProxy`会对方法调用做些判断，但是最终调用的是`MapperMethod`的方法

        
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
        
          if (Object.class.equals(method.getDeclaringClass())) {
        
            return method.invoke(this, args);
        
          } else if (isDefaultMethod(method)) {
        
            return invokeDefaultMethod(proxy, method, args);
        
          }
        
        } catch (Throwable t) {
        
          throw ExceptionUtil.unwrapThrowable(t);
        
        }
        
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        
        return mapperMethod.execute(sqlSession, args);
        }

        

4.`MapperMethod`中有2个成员变量

- SqlCommand类型的command成员
该变量在构造的时候会通过当前接口调用的class名称和method名称在`Configuration`类中寻找对应的缓存。

        //这里是绑定xml声明与与方法的关键部分
        public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
          final String methodName = method.getName();
          final Class<?> declaringClass = method.getDeclaringClass();
          MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
              configuration);
          if (ms == null) {
            if (method.getAnnotation(Flush.class) != null) {
              name = null;
              type = SqlCommandType.FLUSH;
            } else {
              throw new BindingException("Invalid bound statement (not found): "
                  + mapperInterface.getName() + "." + methodName);
            }
          } else {
            name = ms.getId();
            type = ms.getSqlCommandType();
            if (type == SqlCommandType.UNKNOWN) {
              throw new BindingException("Unknown execution method for: " + name);
            }
          }
        }

        //根据class.method寻找
        private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
            Class<?> declaringClass, Configuration configuration) {
          String statementId = mapperInterface.getName() + "." + methodName;
          if (configuration.hasStatement(statementId)) {
            return configuration.getMappedStatement(statementId);
          } else if (mapperInterface.equals(declaringClass)) {
            return null;
          }
          for (Class<?> superInterface : mapperInterface.getInterfaces()) {
            if (declaringClass.isAssignableFrom(superInterface)) {
              MappedStatement ms = resolveMappedStatement(superInterface, methodName,
                  declaringClass, configuration);
              if (ms != null) {
                return ms;
              }
            }
          }
          return null;
        }

- MethodSignature类型的method成员则负责把接口的参数与mapper xml的参数对应起来。 
在上面2个成员变量的初始化都完成过后， 对方法的调用就很简单了:

        //节选
        public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        switch (command.getType()) {
          case INSERT: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
          }
          case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }




特别注意` sqlSession.insert(command.getName(), param)`这段代码。它意味着 ，对接口的方法的调用最终还是回归到了mapper xml的调用。 这里根据解析出来的不同的命令类型分别调用`SqlSession`提供的不同的增删改查接口，然后包装下返回结果就完成了。


所以说`SqlSession`的`insert,update,delete`接口又是如何实现的呢？  配置文件解析又是怎么做的？ 下篇再分析。





