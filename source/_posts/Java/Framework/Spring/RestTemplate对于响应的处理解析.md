---
title: RestTemplate踩坑记录
date: 2019/4/4
updated: 2019/4/19
tags:
categories:
   - Programming Language
   - Java
   - Framework
   - spring
---
作为`Spring Boot`中最常使用的的HTTP客户端，`RestTemplate`在各种Http通讯中都大量使用。
但是，对其原理缺不够熟悉。本文记录的是一次踩坑记录。

<!--more-->

## 问题复现
出现问题的代码类似这样：
```java
//通过RestTemplate给大汉三通短信网关发出请求，已发送短信
private SmsSinglePushResult doSendSms(String url, Object mobile, Object content, Map params){
    try {
        log.info("开始调用大汉三通的接口给用户:{} 发送短信:{}", mobile, content);
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setAccept(Arrays.asList(MediaType.APPLICATION_JSON));
        headers.setAcceptCharset(Arrays.asList(Charset.forName("utf-8")));
        HttpEntity<Map> strEntity = new HttpEntity<>(params, headers);
        //这里报错
        DaHanSanTongResult strResult1 = restTemplate.postForObject(url, strEntity, DaHanSanTongResult.class);

    }catch (Exception e){
        log.error("调用大汉三通的发送短信接口出错。接收手机:"+mobile+" 内容:"+content, e);
    }
}
```
上述例子在项目中被大量使用，几乎没有遇到过问题。`RestTemplate`的第3个参数指定了响应的类型被反序列化的类型。而在大汉三通的文档中也明确写到请求和响应都是`UTF-8`格式的`JSON`字符串，所以想当然的认为这样是没有问题的。
实际运行结果如下：

```java
org.springframework.web.client.RestClientException: Could not extract response: no suitable HttpMessageConverter found for response type [class com.ctspcl.sms.gateway.manager.sms.dahansantong.base.DaHanSanTongResult] and content type [application/octet-stream]
    at org.springframework.web.client.HttpMessageConverterExtractor.extractData(HttpMessageConverterExtractor.java:110)
    at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:655)
    at org.springframework.web.client.RestTemplate.execute(RestTemplate.java:613)
    at org.springframework.web.client.RestTemplate.postForObject(RestTemplate.java:380)
```

## 问题追踪
上面的问题很明显，对于响应头`application/octet-stream`找不到对应的解析器。这里奇怪的是大汉三通的文档中明确写到响应是`JSON`格式字符串，但是为什么响应的`Content-Type`是
`application/octet-stream`这个较为少见的头呢？  于是抓包看下：
### 抓包
请求抓包:
![请求抓包](http://p5rx80fa6.bkt.clouddn.com/%E8%AF%B7%E6%B1%82%E5%8F%91%E5%87%BA.png)
响应抓包:
![响应抓包](http://p5rx80fa6.bkt.clouddn.com/%E8%AF%B7%E6%B1%82%E5%93%8D%E5%BA%94.png)

可以看到，响应中`HTTP`头事实上是没有`Content-Type`字段的。那么这个`application/octet-stream`是哪里来的？ 是因为我们请求时指定的返回`Bean`影响了这个头么？

看下代码。

### 源码分析
通过源码发现，`RestTemplate`把对响应结果的处理封装为一个`HttpMessageConverterExtractor`。然后在获取到响应后调用其中的

```java
//大部分用户调用的接口的实现都类似这样
public <T> T postForObject(String url, Object request, Class<T> responseType, Object... uriVariables)
    throws RestClientException {
    RequestCallback requestCallback = httpEntityCallback(request, responseType);
    HttpMessageConverterExtractor<T> responseExtractor =
        new HttpMessageConverterExtractor<T>(responseType, getMessageConverters(), logger);
    return execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables);
}
//对于包含URL参数的请求的处理
public <T> T execute(String url, HttpMethod method, RequestCallback requestCallback,
                         ResponseExtractor<T> responseExtractor, Object... uriVariables) throws RestClientException {
    URI expanded = getUriTemplateHandler().expand(url, uriVariables);
    return doExecute(expanded, method, requestCallback, responseExtractor);
}
//实际处理部分
protected <T> T doExecute(URI url, HttpMethod method, RequestCallback requestCallback,
                          ResponseExtractor<T> responseExtractor) throws RestClientException {
    ClientHttpResponse response = null;
    try {
        ClientHttpRequest request = createRequest(url, method);
        if (requestCallback != null) {
            requestCallback.doWithRequest(request);
        }
        response = request.execute();
        handleResponse(url, method, response);
        if (responseExtractor != null) {
            //解析响应结果
            return responseExtractor.extractData(response);
        }
        else {
            return null;
        }
    }
    catch (IOException ex) {
    //忽略异常
    }

}
```

由于通过抓包我们可以确定Http响应是没有`Content-Type`，所以基本可以确定`application/octet-stream`这个头是在这个`Extractor`中加上的。查看下这个方法：
```java
public T extractData(ClientHttpResponse response) throws IOException {
    MessageBodyClientHttpResponseWrapper responseWrapper = new MessageBodyClientHttpResponseWrapper(response);
    if (!responseWrapper.hasMessageBody() || responseWrapper.hasEmptyMessageBody()) {
        return null;
    }
    //根据响应获取ContentType
    MediaType contentType = getContentType(responseWrapper);
    for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
        if (messageConverter instanceof GenericHttpMessageConverter) {
            GenericHttpMessageConverter<?> genericMessageConverter =
                (GenericHttpMessageConverter<?>) messageConverter;
            if (genericMessageConverter.canRead(this.responseType, null, contentType)) {
                return (T) genericMessageConverter.read(this.responseType, null, responseWrapper);
            }
        }
        if (this.responseClass != null) {
            if (messageConverter.canRead(this.responseClass, contentType)) {
                return (T) messageConverter.read((Class) this.responseClass, responseWrapper);
            }
        }
    }
}
```

```java
private MediaType getContentType(ClientHttpResponse response) {
    MediaType contentType = response.getHeaders().getContentType();
    if (contentType == null) {
        //这里设置了默认的ContentType
        contentType = MediaType.APPLICATION_OCTET_STREAM;
    }
    return contentType;
}
```

上面可以看到代码中设置了默认的`Content-Type`为`MediaType.APPLICATION_OCTET_STREAM`，也就是`application/octet-stream`。 所以说，响应头变为这个也就可以解释了。
但是还有另外一个疑惑： 平常（`Spring Boot`项目）在这样使用
```
restTemplate.postForObject(url, strEntity, DaHanSanTongResult.class);
```
的时候，为什么可以直接把响应转换为一个具体的POJO。服务器的响应都是`JSON`字符串，也都是`UTF-8`格式。通常，上述也可以通过返回`String`来处理
```
restTemplate.postForObject(url, strEntity, String.class);
```
这两种都可以正常处理，而上面这种情况却无法处理。

上面代码中的`extractData`方法很明显是问题的关键。其中，遍历了当前所有的`HttpMessageConverter`，然后挨个调用其中的`canRead`方法。 该方法的签名:
```java
boolean canRead(Type type, Class<?> contextClass, MediaType mediaType);
```
可以看到，这个`canRead`方法接收一个目标类(`Class`)类型，一个上下文类型（不重要）， 以及一个`MediaType`。 这也就是说，这个方法决定了** 当前这个`HttpMessageConverter`支持哪种`MediaType`转换为哪种`Class`**。

查看当前这个`Extractor`,发现它支持如下的`HttpMessageConverter`(默认的`Spring Boot`配置，没有另外添加):
- `ByteArrayHttpMessageConverter`
- `StringArrayHttpMessageConverter`
- `ResourceHttpMessageConverter`
- `SourceHttpMessageConverter`
- `AllEncompassingFormHttpMessageConverter`
- `Jaxb2RootElementHttpMessageConverter`
- `MappingJackson2HttpMessageConverter`

对每个类查看，发现他们对于`MediaType`和`Class`的支持情况如下:
- `ByteArrayHttpMessageConverter`
支持`application/octet-stream, */*`的`MediaType`与`byte[]`的`Class`
- `StringArrayHttpMessageConverter`
支持`application/text/plain, */*`的`MediaType`与`String`的`Class`
- `ResourceHttpMessageConverter`
支持`*/*`的`MediaType`与`Resource`的`Class`
- `SourceHttpMessageConverter`
支持`application/xml,text/xml`的`MediaType`与`DOMSource，SAXSource，StAXSource，StreamSource，Source`的`Class`
- `AllEncompassingFormHttpMessageConverter`
这是一个复合类，它会根据当前的`ClassLoader`加载如下的一个`HttpMessageConverter`:
 - Jaxb2RootElementHttpMessageConverter
 - MappingJackson2HttpMessageConverter
 - GsonHttpMessageConverter
 - MappingJackson2XmlHttpMessageConverter
- `Jaxb2RootElementHttpMessageConverter`
支持`application/xml,text/xml,application/+xml`的`MediaType`与被`XmlRootElement，XmlType`注解的`Class`。 这是一个使用`JAXB2`的`Converter`，对于`XML`格式的响应会比较常用
- `MappingJackson2HttpMessageConverter`
支持`application/json，application/+json`的可以被`JackSon`序列化的`Class`。

### 结论及解决方式
所以在类似调用`restTemplate.postForObject(url, strEntity, XXXBean.class);`的时候，如果Http响应头的`Content-Type`为`application/json，application/+json`就使用`JackSon`来反序列化Http Body为对应的Java类。但是上面我们遇到的问题由于响应头不是这个，并且期望的返回类也不是上面7个`HttpMessageConverter`所支持的`Class`中任何一个，自然会报**```no suitable HttpMessageConverter found```** 这种错误。而我们发现`String, byte[]`的`Class`是可以被序列化的，所以改为其中任意一个即可解决该问题。
同时，正常情况下服务器对应响应是应该设置*正确*的`Content Type`的，上面这个服务器的行为对于用户就很不友好了。

### 后续
本文没有对`RestTemplate`对于请求的处理解析。关键逻辑几乎都在`RestTemplate`的内部类`HttpEntityRequestCallback`的`doWithRequest`方法中，一看就懂。内部逻辑与解析响应雷同。

这里也感慨下程序员的技术道路，现在有很多的公众号或者app提供了很多文章，对很多东西都有了总结性的介绍，但是如果一个开发者仅仅看这些，不深入到书本、深入到代码中去研究，是如同无根浮萍的。 往远了说，大厂为什么青睐招985/211的人，即使有些代码本身写的并不好。 其实也就是一个根基问题，而根基决定了可塑性。
