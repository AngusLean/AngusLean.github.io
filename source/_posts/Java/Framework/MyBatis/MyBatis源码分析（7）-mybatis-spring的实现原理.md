---
title: MyBatis源码分析（7）-mybatis-spring的实现原理
date: 2018/1/2
updated: 2018/1/25
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
`mybatis-spring`把`mybatis`与`spring`无缝连接，它允许 `mybatis`嵌入到spring的事务，处理 `mybatis`的mapper实例创建以及注入到其他Bean中， 翻译mybatis的异常到spring的`DataAccessExceptions`。
鉴于`mybatis-spring`的使用范围以及`spring`的普集程度，所以把`mybatis-spring`的源码分析放到同一系列中来。
我觉得说`mybatis-spring`至少有如下几个问题值得探究：

<!--more-->

- 是如何接管mybatis默认的使用`sqlSessionFactory`获取mapper的？
- 如何与`spring`协作的？
- 如何解决前面说的多个session同时存在时缓存的问题？

下面将一一剖析这几个问题。 

### 1.`mybatis`使用回顾与`mybatis-spring`使用
首先看下`mybatis`使用：
```java
//创建一个sqlsessionfactory作为使用工厂
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory =
    new SqlSessionFactoryBuilder().build(inputStream);
//通过sqlsessionfactory获取接口
SqlSession session = sqlSessionFactory.openSession();
try {
    Blog blog = session.selectOne(
        "org.mybatis.example.BlogMapper.selectBlog", 101);
} finally {
    //关闭接口
    session.close();
}
```

主要是如下3步：
- 根据配置文件构建一个`SqlSessionFactory`
- 通过`SqlSessionFactory`获取一个`SqlSession`
- 通过`SqlSession`获取接口实例或者mapper name操作数据库

再看以下使用`mybatis-spring`的一般方式：
直接在spring的配置文件中增加如下配置：
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="configLocation" value="classpath:config/mybatis.xml"/>
    <property name="mapperLocations" value="classpath:sqlmap/*.xml"/>
</bean>

<!-- 手动实现DAO,需要得到sqlSession -->
<bean id="sqlSessionTemplate"
      class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg index="0" ref="sqlSessionFactory" />
</bean>

<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="xxx.xxx.xxx"/>
    <property name="sqlSessionTemplateBeanName" value="sqlSessionTemplate" />
</bean>

```

然后在实际使用中一般使用`Resource`或者`Autowired`注解获取上面配置的`basePackage`中的某个接口名就可以了。 应用不再需要关系mybatis通过配置文件初始化以及获取`SqlSessionFactory`这些步骤，不需要手动获取mapper，也不需要自己管理生命周期、事务。


### 2.Mapper接口的注册与从Spring获取
在使用`mybatis`时，获取Mapper接口需要通过`SqlSession`接口，而在spring中是通过注入自动获取的，是如何实现的呢？

首先看下整体流程的结构图：

##### 获取mapper实例：
![获取mapper接口](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-spring-get-mapper-instance.jpg)

#### 注册mapper：
![注册mapper接口](https://gitee.com/angus_lean/markDownPic/raw/master/2017/11/mybatis-spring-registry.jpg)

追本溯源，spring不可能无缘无故就找到我们的mapper接口，肯定是在某个配置中指定的。 而使用mybatis-spring需要配置的只有上面几个， 其中与mapper有关的只有这个：
```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="xxx.xxx.xxx"/>
    <property name="sqlSessionTemplateBeanName" value="sqlSessionTemplate" />
</bean>
```

#### 2.1 `MapperScannerConfigurer`
这里给类`MapperScannerConfigurer`配置了2个属性：`basePackage，sqlSessionTemplateBeanName`。 第一个是mapper 的全限定类名，而第二个则是mybatis-spring的一个类。 我们先看看这个`MapperScannerConfigurer`。
这个类的注释中开头是这样的：
>BeanDefinitionRegistryPostProcessor that searches recursively starting from a base package for interfaces and registers them as {@code MapperFactoryBean}. Note that only interfaces with atleast one method will be registered; concrete classes will be ignored.
PS: 该类继承自BeanDefinitionRegistryPostProcessor

上面的注释说明这个`MapperScannerConfigurer`做的事情就是递归搜索所有接口并且把它们注册为一个`MapperFactoryBean`。

#### 2.2 `MapperFactoryBean`
该类的核心方法是`postProcessBeanDefinitionRegistry`:
```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
        processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

#### 2.3 `ClassPathMapperScanner`
这里就是需要被扫描包并且添加到`BeanDefinitionRegistry`的逻辑，可以看到实际处理被转交给`ClassPathMapperScanner`，该类中的`Scan`为一个父类方法，实际会调用子类的`doScan`方法。我们看下实现：
```java
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
//当前类是继承自ClassPathBeanDefinitionScaner,调用它的扫描方法
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
    if (beanDefinitions.isEmpty()) {
    } else {
        processBeanDefinitions(beanDefinitions);
    }
    return beanDefinitions;
}

private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    //遍历当前扫描到的所有的mapper的定义，然后做一些修改。
    for (BeanDefinitionHolder holder : beanDefinitions) {
        definition = (GenericBeanDefinition) holder.getBeanDefinition();
        //实际的bean类名，也就是我们的Mapper类名
        String beanClassName = definition.getBeanClassName();
        // the mapper interface is the original class of the bean
        // but, the actual class of the bean is MapperFactoryBean
        definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName);
        //将类修改为mapperFactoryBean
        definition.setBeanClass(this.mapperFactoryBean.getClass());
        definition.getPropertyValues().add("addToConfig", this.addToConfig);
        boolean explicitFactoryUsed = false;
        //增加属性： sqlSessionFactory或者sqlSessionTemplate，这个我们配置中有
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
            definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
            definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
            explicitFactoryUsed = true;
        }

        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
            definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
            definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
            explicitFactoryUsed = true;
        }
        if (!explicitFactoryUsed) {
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }
    }
}
```

#### 2.4 `MapperFactoryBean`
上面的代码说明的问题是，最终生成Bean实例的是在`MapperFactoryBean`中。 我们再看看这个类的实现：
```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

    protected void checkDaoConfig() {
        super.checkDaoConfig();
        //这里事实上和Mybatis原生的方式一样，只不过mybatis是sqlsessionfactorybuilder在做这个事情。
        Configuration configuration = getSqlSession().getConfiguration();
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                //这里添加到全局配置中
                configuration.addMapper(this.mapperInterface);
            } catch (Exception e) {
                throw new IllegalArgumentException(e);
            } finally {
                ErrorContext.instance().reset();
            }
        }
    }
    //获取一个Mapper实例
    @Override
        public T getObject() throws Exception {
        //最终还是通过SqlSession获取
        return getSqlSession().getMapper(this.mapperInterface);
    }
    //单例
    public boolean isSingleton() {
    return true;
  }
}
```
在这里看到`MapperFactoryBean`接口实现了`FactoryBean`接口，该接口是一个生成Bean实例的工厂类， `MapperFactoryBean`类的`getObject`方法实际上调用了`MapperFactoryBean`的父类`SqlSessionDaoSupport`中的`getSqlSession()`方法，从这个方法返回的实例中获取Mapper. 这里`getMapper(Class)`与使用`Mybatis`是完全一样的。 只不过`Mybatis`的`SqlSession`的默认实现是`DefaultSqlSession`，然后这个类再调用`Executor`类处理逻辑，这里肯定不一样

#### 2.5 `SqlSessionDaoSupport`

我们看下`SqlSessionDaoSupport`这个类：

```java
public abstract class SqlSessionDaoSupport extends DaoSupport {

    private SqlSessionTemplate sqlSessionTemplate;

    public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
        if (this.sqlSessionTemplate == null || sqlSessionFactory != this.sqlSessionTemplate.getSqlSessionFactory()) {
            this.sqlSessionTemplate = createSqlSessionTemplate(sqlSessionFactory);
        }
    }
    
    protected SqlSessionTemplate createSqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    public final SqlSessionFactory getSqlSessionFactory() {
        return (this.sqlSessionTemplate != null ? this.sqlSessionTemplate.getSqlSessionFactory() : null);
    }
    //返回成员变量sqlSessionTemplate
    public SqlSession getSqlSession() {
        return this.sqlSessionTemplate;
  }
```
所以这里的`SqlSession`的实现是`SqlSessionTemplate`， 构造方法中传入了`SqlSessionFactory`。

#### 2.6 `SqlSessionTemplate`

通过上面我们看到最终的`SqlSession`的类型是`SqlSessionTemplate`。该类实际上就是我们在使用`Mybatis`时使用的`SqlSession`的默认实现`DefaultSqlSession`在`Mybatis-Spring`中的包装类。`SqlSessionTemplate`提供了如下的功能：

- select/insert/update/delete
- 获取Mapper实例

一般情况下，我们不会直接使用这个类而是依赖于sping的自动注入，而sping的自动注入的流程在上面已大概介绍了，最终生成Mapper实例还是会走到`SqlSessionTemplate`中来。如果需要直接在我们的应用中调用mapper xml中的方法（也就是`SqlSession`的`selectOne,update,delete`等方法）,同时，又希望spring来管理事务以及声明周期的话，我们就可以在配置文件中配置一个`SqlSessionTemplate`的Bean,指定一个`SqlSessionFactory`参数即可。

我们看下这个类的说明：

> Thread safe, Spring managed, {@code SqlSession} that works with Spring
transaction management to ensure that that the actual SqlSession used is the
one associated with the current Spring transaction. In addition, it manages
the session life-cycle, including closing, committing or rolling back the
session as necessary based on the Spring transaction configuration.

> The template needs a SqlSessionFactory to create SqlSessions, passed as a
constructor argument. It also can be constructed indicating the executor type
to be used, if not, the default executor type, defined in the session factory
will be used.

> This template converts MyBatis PersistenceExceptions into unchecked
DataAccessExceptions, using, by default, a  MyBatisExceptionTranslator.

注释主要有以下几点意思：
- 线程安全，Spring管理。保证当前的`SqlSession`是和当前spring事务关联的。 这个类管理了整个session的声明周期，包括关闭，创建，提交或者回滚
- 构造时需要一个SqlSessionFactory
- 转换mybatis的异常为不受检的异常。通过MyBatisExceptionTranslator类。

看下这个类的实现：
```java
public class SqlSessionTemplate implements SqlSession, DisposableBean{
    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,PersistenceExceptionTranslator exceptionTranslator) {
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
        //实际上处理逻辑的是一个SqlSessionFactory的代理类。
        this.sqlSessionProxy = (SqlSession) newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[] { SqlSession.class },
            new SqlSessionInterceptor());
    }
    //常见的mapper xml调用
    public <T> T selectOne(String statement) {
        return this.sqlSessionProxy.selectOne(statement);
    }
    //获取mapper实例
    public <T> T getMapper(Class<T> type) {
        return getConfiguration().getMapper(type, this);
    }
    public Configuration getConfiguration() {
        return this.sqlSessionFactory.getConfiguration();
    }
    //代理类，包装了SqlSessionFactory
    private class SqlSessionInterceptor implements InvocationHandler {
        @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            SqlSession sqlSession = getSqlSession(
                SqlSessionTemplate.this.sqlSessionFactory,
                SqlSessionTemplate.this.executorType,
                SqlSessionTemplate.this.exceptionTranslator);
            try {
                Object result = method.invoke(sqlSession, args);
                if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    // force commit even on non-dirty sessions because some databases require
                    // a commit/rollback before calling close()
                    sqlSession.commit(true);
                }
                return result;
            } catch (Throwable t) {
                Throwable unwrapped = unwrapThrowable(t);
                if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                    closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                    sqlSession = null;
                    Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
                    if (translated != null) {
                        unwrapped = translated;
                    }
                }
                throw unwrapped;
            } finally {
                if (sqlSession != null) {
                    closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                }
            }
        }
    }
}
```

可以看到，`SqlSessionTemplate`只是使用`SqlSessionInterceptor`代理了一个`SqlSessionFactory`，通过一个叫做`PersistenceExceptionTranslator`的类转换mybatis异常到spring异常。
`SqlSessionInterceptor`的代理逻辑如下：
1. 通过sqlsessionfactory, executortype, persistenceExceptionTranslator构造一个`SqlSession`实例
2. 调用该实例的某个方法
3. 根据sqlsessionfactory以及sqlsession实例判断是否需要提交（commit）
4. 发生任何异常都使用`PersistenceExceptionTranslator`包装异常然后返回
5. 关闭`SqlSession`实例

所以，分析到这里，再结合上面的结构图，mybatis-spring的mapper注册与获取就是一目了然了。


