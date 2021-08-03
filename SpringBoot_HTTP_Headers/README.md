# SpringBoot修改通用Http Headers

## 背景:

>HTTP的response headers的Date是GMT，时间比东八区(国内)时间差八个小时

> 这个问题其实应该归结于[rfc7231](https://tools.ietf.org/html/rfc7231#section-7.1.1.1)设定的标准，只让用GMT/UTC标准时间。<br>

> 坑爹的是原本的标准是要加时区的，但被军方给用错了，所以就把时区当成错字去掉了。[相关链接](https://tools.ietf.org/html/rfc1123#page-55 )

这里建议查看相关日志的同学忽略Date的时间。但问题总有解决的办法，如果准确的时间很重要，且有其它的Header可能需要修改，那么请往后看。

JavaWeb中修改HTTP Headers很容易，只要在响应请求的逻辑中获取`HttpServletResponse`后`setHeader`。

但关于修改`Date`，`Server`这类默认的header，总是在业务逻辑中`setHeader`显然不适合维护。以下是我分析Tomcat以及SpringBoot处理请求的整个流程后得出的修改header的理想方法。

如果不想看原理分析，请直接看[推荐方法](#conclusion)

## 分析：

### Tomcat处理请求流程：
![image](https://github.com/lclmichael/knowledge/blob/master/SpringBoot_HTTP_Headers/tomcat-rquest-flow.png?raw=true)
> 1. 用户点击网页内容，请求被发送到本机端口8080，被在那里监听的Coyote HTTP/1.1 Connector获得。 
> 1. Connector把该请求交给它所在的Service的Engine来处理，并等待Engine的回应。 
> 1. Engine获得请求localhost/test/index.jsp，匹配所有的虚拟主机Host。 
> 1. Engine匹配到名为localhost的Host（即使匹配不到也把请求交给该Host处理，因为该Host被定义为该Engine的默认主机），名为localhost的Host获得请求/test/index.jsp，匹配它所拥有的所有的Context。Host匹配到路径为/test的Context（如果匹配不到就把该请求交给路径名为“ ”的Context去处理）。 
> 1. path=“/test”的Context获得请求/index.jsp，在它的mapping table中寻找出对应的Servlet。Context匹配到URL PATTERN为*.jsp的Servlet,对应于JspServlet类。 
> 1. 构造HttpServletRequest对象和HttpServletResponse对象，作为参数调用JspServlet的doGet（）或doPost（）.执行业务逻辑、数据存储等程序。 
> 1. Context把执行完之后的HttpServletResponse对象返回给Host。 
> 1. Host把HttpServletResponse对象返回给Engine。 
> 1. Engine把HttpServletResponse对象返回Connector。 
> 1. Connector把HttpServletResponse对象返回给客户Browser。

`Connector`监听端口后的操作2-9都是通过`Processor`处理请求，这里有一个[UML序列流程图](http://tomcat.apache.org/tomcat-7.0-doc/architecture/requestProcess/request-process.png)。图较大，请自行放大查看。


在整个流程中，只有一处涉及到`Date`的修改：

``` java
`apache-tomcat-7.0.82-src`
package org\apache\coyote\http11;
public abstract class AbstractHttp11Processor<S> extends AbstractProcessor<S> {
    ...
    private void prepareResponse() {
        ...
        if (headers.getValue("Date") == null) {
            headers.setValue("Date").setString(FastHttpDateFormat.getCurrentDate());
        }
        ...
    }
    ...
}
```
其它的服务器默认HTTP Headers也几乎是在此设置，除了`Server`属性(就是服务器名称)外没有可以根据配置设置的办法。修改Tomcat源码再编译代价太大，最好不要在服务器层级修改。

### SpringBoot处理请求流程：


![image](https://github.com/lclmichael/knowledge/blob/master/SpringBoot_HTTP_Headers/spring-mvc-rquest-flow.png?raw=true)

SpringBoot在Tomcat处理请求的流程中，处在`Servlet`层级，在SpringMVC处理完请求后，仍然要通过从`Context`传出。上图中的Front controller是SpringMVC的DispatcherServelet统一拦截器，SpringBoot默认使用SpringMVC处理web请求。在代码中的流程图：

![image](https://github.com/lclmichael/knowledge/blob/master/SpringBoot_HTTP_Headers/springboot-request-process.png?raw=true)

整个SpringBoot对Headers的操作同样很少，而且几乎都是在消息转换器`HttpMessageConverters`中实现的，根据转换的类型最终生成相应的`Content-Type`。

在`org.springframework.http.converter.AbstractHttpMessageConverter addDefaultHeaders`有相关实现。 

所以SpringBoot中没有设置Headers的相关配置。然而自定义消息转换器对框架侵入太大。修改Headers最好在拦截器`postHandle`中进行。


## 环境：

> JDK8

> Tomcat7

> SpringBoot1.5

## 修改原则：

1. 基于Server (Nginx / Tomcat)或者SpringBoot原生配置修改
1. 侵入度最低的代码配置，不动框架主体，不混入业务逻辑，松耦合

## <p id="conclusion">推荐方法</p>

1. 配置方法：`{Tomcat目录}/conf/server.xml`在`context`中增加`server`属性

```
<Connector port="80" protocol="HTTP/1.1" server="customize name">
```

2. 其它需要自定义Headers：

根据`ServletFilter`或者`SpringBoot Inceptor`来实现，这里给出`Inceptor`例子：
```
@Configuration
public class ResponseHeaderConfig extends WebMvcConfigurerAdapter {

    @Bean
    public ResponseHeaderInterceptor responseHeaderInterceptor() {
        return new ResponseHeaderInterceptor();
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(responseHeaderInterceptor()).addPathPatterns("/*");
        super.addInterceptors(registry);
    }
}

@Component
public class ResponseHeaderInterceptor implements HandlerInterceptor {

    //...省略其它接口实现

    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest,
            HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
                //这里设置
                //response.setHeader("Date", time);
            }
}
```


