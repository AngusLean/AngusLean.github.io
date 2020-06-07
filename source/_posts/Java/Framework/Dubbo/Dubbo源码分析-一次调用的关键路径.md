---
title: Dubbo的一次RPC调用客户端关键路径分析
date: 2019/05/07
updated: 2019/05/07
tags:
   - Dubbo
   - RPC
   - 分布式
categories:
   - Programming Language
   - Java
   - Framework
   - Dubbo
---

Dubbo作为阿里开源的RPC框架，它不仅仅包含了传统的RPC功能，得益于自身良好的架构设计, Dubbo还在服务治理等方面也颇有功力。例如路由、限流、熔断、降级等等都可以轻易的在上面实现。在国内，Dubbo和Sping Boot毫无疑问是最为广泛使用的两个框架（某种程度上说并不对等）。本系列文章将尝试理清Dubbo自身的一些关键设计以及一些扩展点。

<!--more-->

### 整体架构
在集群使用时，dubbo的调用流程如下：
![整体架构](http://www.ideabuffer.cn/2018/05/13/Dubbo%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%9ADirectory/cluster.jpg)
根据[dubbo官网](http://dubbo.incubator.apache.org/en-us/docs/dev/design.html)的介绍，他们的核心概念如下：
- `Invoker`, Dubbo领域对象中的entity对象，它代表一个调用实体（可执行）。你可以对一个`Invoker`执行调用。它的具体实现可能是本地的、也可能是远程实现，或者是一个`Cluster`。
- `Directory`, 是`Invoker`的注册目录，换句话说，就是一个`List<Invoker>`。这个List既可以是本地写死的，也可以是动态变化的（根据不同的注册中心，比如`Zookeeper`）
- `Router`,路由组件。用于根据请求从`Directory`过滤出多个*具体可用*的`Invoker`
- `LoadBalance`, 负载均衡。负责从上面路由过后的多个`Invoker`中选出一个具体的`Invoker`
- `Cluster` 多个`Invoker`的抽象，使得多个 `Invoker`表现的和一个一样。

从架构图上说，Cluster代表多个`Invoker`, 这些`Invoker`来自于`Directory`,而`Directory`中的数据则来自于注册中心（通常不会是静态的），然后经过`Router, LoadBalance`选出其中可用的、最合适的一个`Invoker`进入协议层使用。 调用链图如下:
![调用链](http://dubbo.incubator.apache.org/docs/en-us/dev/sources/images/dubbo-extension.jpg)


### 调用链
我们通过官网的[消费者Demo](http://dubbo.incubator.apache.org/en-us/docs/user/quick-start.html) 来查看:
```
public class Consumer {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"spring/consumer.xml"});
        context.start();
        // Obtaining a remote service proxy
        DemoService demoService = (DemoService)context.getBean("demoService");
        // Executing remote methods
        String hello = demoService.sayHello("world");
        // Display the call result
        System.out.println(hello);
    }
}

```
在调用RPC方法`sayHello`时通过一次转发进入`MockClusterInvoker`中：
```
public Result invoke(Invocation invocation) throws RpcException {
    Result result = null;

    String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
    if (value.length() == 0 || value.equalsIgnoreCase("false")) {
        //正常调用走这里，进入AbstractClusterInvoker
        result = this.invoker.invoke(invocation);
    } else if (value.startsWith("force")) {
        result = doMockInvoke(invocation, null);
    } else {
        try {
            result = this.invoker.invoke(invocation);
        } catch (RpcException e) {
        }
    }
    return result;
}
```
`MockClusterInvoker`主要是进行本地伪装以及Mock使用（测试或者故障处理），这个后续再说。这里的调用进一步进入`AbstractClusterInvoker`中:
```
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();

    // binding attachments into invocation.
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addAttachments(contextAttachments);
    }
    //从Directory中获取所有可用的Invoker
    List<Invoker<T>> invokers = list(invocation);
    //获取负载均衡器
    LoadBalance loadbalance = initLoadBalance(invokers, invocation);
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    //实际调用
    return doInvoke(invocation, invokers, loadbalance);
}

```

前面说过，`Cluster`主要就是伪装多个`Invoker`,上面代码可以看到这些。其中，最为重要的如下三步:
- 从`Directory`中获取可用的`Invoker`
- 创建负载均衡
- 实际调用

#### `Directory`本地注册中心
`list`方法最终进入了`RegistryDiretory`中的方法:
```
public List<Invoker<T>> doList(Invocation invocation) {
    if (multiGroup) {
        return this.invokers == null ? Collections.emptyList() : this.invokers;
    }
    List<Invoker<T>> invokers = null;
    try {
        //从过滤器链中获取可用的Invoker列表
        invokers = routerChain.route(getConsumerUrl(), invocation);
    } catch (Throwable t) {
    }
    return invokers == null ? Collections.emptyList() : invokers;
}

```
而`RouterChain`中的逻辑则是:
```
public List<Invoker<T>> route(URL url, Invocation invocation) {
//来自于注册中心的invoker缓存
    List<Invoker<T>> finalInvokers = invokers;
//熟悉的责任链模式
    for (Router router : routers) {
        finalInvokers = router.route(finalInvokers, url, invocation);
    }
    return finalInvokers;
}
```
该方法仅仅是对`RouterChain`中的invoker走了一遍过滤器。对于过滤器我们后续列出，这里关注这个`invokers`变量是如何刷新的。

#### 远程实例如何获取
这里说的“远程实例”就是指上面的`invokers`.在Dubbo中，`RegistryDirectory`就是代表着本地保存的所有远程服务列表，而前面我们说过，Dubbo通过注册中心来**注册**服务，必然
有某种*“方式”*通知`RegistryDirectory`。同时，在服务挂掉或者重新上线时，注册中心总是知道的，着也需要通知到`RegistryDirectory`。
这里的秘诀就是`NotifyoListener`接口。该接口只有一个方法:
```
public interface NotifyListener {

   //dubbo通过URL总线的方式在不同层传递数据，可以参考官方文档。
    void notify(List<URL> urls);
}

```
而`RegistryDirectory`实现了`NotifyListener`接口:
```
public synchronized void notify(List<URL> urls) {
    //这里添加了Router类，与invoker逻辑无关。
    List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
    this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);
    List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
    toRouters(routerURLs).ifPresent(this::addRouters);

    // 解析出服务提供者列表
    List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
    refreshOverrideAndInvoker(providerURLs);
}

//上面refreshOverrideAndInvoker会进入到这里
private void refreshInvoker(List<URL> invokerUrls) {
    //忽略几个空判断逻辑
    this.forbidden = false; // Allow to access
    Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
    if (invokerUrls == Collections.<URL>emptyList()) {
        invokerUrls = new ArrayList<>();
    }
    if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
        invokerUrls.addAll(this.cachedInvokerUrls);
    } else {
        this.cachedInvokerUrls = new HashSet<>();
        this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
    }
    //将URL地址转换为Invokern
    Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map
    List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
    routerChain.setInvokers(newInvokers);
}

```
然后就调用到`RouterChain`中:
```
public void setInvokers(List<Invoker<T>> invokers) {
//这里就保存了所有的远程服务提供者信息。
    this.invokers = (invokers == null ? Collections.emptyList() : invokers);
    routers.forEach(router -> router.notify(this.invokers));
}
```
这样就是一个完整的刷新本地服务缓存的流程（实际的注册中心调用在后面的`Registry`再分析）
#### 负载均衡
从上面的逻辑中我们看到，在`RegistryDirectory`中拿到注册中心通知过来的`Invoker`过后，经过了过滤器链。然后紧接者就是通过负载均衡在一系列的**可用**的远程服务中选择
具体的一个了。该步骤主要有亮点：
- 得到一个负载均衡实例
- 选择具体的`Invoker`
获取`LoadBalance`实例比较有Dubbo的风格：通过SPI拿到：
```
protected LoadBalance initLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {
    if (CollectionUtils.isNotEmpty(invokers)) {
    //dubbo自定义的SPI类.Dubbo中很多重要的接口都是通过SPI获取实例的,比如注册中心。
        return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                .getMethodParameter(RpcUtils.getMethodName(invocation), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    } else {
        return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }
}
```
从上面代码可以看出，Dubbo通过`SPI`机制拿到具体的负载均衡策略实例，然后紧接着进入选择逻辑:
```
private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation,
                            List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
if (invokers.size() == 1) {
    return invokers.get(0);
}
    //调用负载均衡实例的选择方法
    Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);
    //如果通过负载均衡选出来的Invoker已经被选择或者是不可达（不保证负载均衡会处理这种情况）又或者有可用性检测
    //If the `invoker` is in the  `selected` or invoker is unavailable && availablecheck is true, reselect.
    if ((selected != null && selected.contains(invoker))
            || (!invoker.isAvailable() && getUrl() != null && availablecheck)) {
        try {
            //重新选择
            Invoker<T> rInvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
            if (rInvoker != null) {
                invoker = rInvoker;
            } else {
                //Check the index of current selected invoker, if it's not the last one, choose the one at index+1.
                int index = invokers.indexOf(invoker);
                try {
                    //Avoid collision
                    invoker = invokers.get((index + 1) % invokers.size());
                } catch (Exception e) {
                    logger.warn(e.getMessage() + " may because invokers list dynamic change, ignore.", e);
                }
            }
        } catch (Throwable t) {
            logger.error("cluster reselect fail reason is :" + t.getMessage() + " if can not solve, you can set cluster.availablecheck=false in url", t);
        }
    }
    return invoker;
}

```
