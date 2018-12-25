## 前言

在微服务架构中，需要几个基础的服务治理组件，包括服务注册与发现、服务消费、负载均衡、断路器、智能路由、配置管理等，由这几个基础组件相互协作，共同组建了一个简单的微服务系统。一个简答的微服务系统如下图：
![](https://upload-images.jianshu.io/upload_images/13532499-7bb1fcab03da37a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在Spring Cloud微服务系统中，一种常见的负载均衡方式是，客户端的请求首先经过负载均衡（zuul、Ngnix），再到达服务网关（zuul集群），然后再到具体的服。，服务统一注册到高可用的服务注册中心集群，服务的所有的配置文件由配置服务管理（下一篇文章讲述），配置服务的配置文件放在git仓库，方便开发人员随时改配置。

## 一、Zuul简介

Zuul的主要功能是路由转发和过滤器。路由功能是微服务的一部分，比如／api/user转发到到user服务，/api/shop转发到到shop服务。zuul默认和Ribbon结合实现了负载均衡的功能。
zuul有以下功能：

- Authentication
- Insights
- Stress Testing
- Canary Testing
- Dynamic Routing
- Service Migration
- Load Shedding
- Security
- Static Response handling
- Active/Active traffic management

  ## 二、准备工作

  在原有的工程上新建一个工程

  ## 三、创建service-zuul工程

  pom依赖如下
 ```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>server-zuul</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>server-zuul</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
</project>
 ```
在其入口applicaton类加上注解@EnableZuulProxy，开启zuul的功能：
```
package com.example.serverzuul;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@EnableDiscoveryClient
@EnableEurekaClient
@EnableZuulProxy
@SpringBootApplication
public class ServerZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServerZuulApplication.class, args);
    }
}
```
加上配置文件application.yml加上以下的配置代码：
```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

server:
  port: 8766

spring:
  application:
    name: service-zuul

zuul:
  routes:
    api-a:
      path: /api-a/**
      serviceId: service-ribbon
    api-b:
      path: /api-b/**
      serviceId: service-feign
```
首先指定服务注册中心的地址为http://localhost:8761/eureka/，服务的端口为8766，服务名为service-zuul；以/api-a/ 开头的请求都转发给service-ribbon服务；以/api-b/开头的请求都转发给service-feign服务；
**坑来了**
依次运行这五个工程;打开浏览器访问：http://localhost:8766/api-a/hi?name=lisi ;浏览器显示：
```
Description:

The bean 'proxyRequestHelper', defined in class path resource [org/springframework/cloud/netflix/zuul/ZuulProxyAutoConfiguration$NoActuatorConfiguration.class], could not be registered. A bean with that name has already been defined in class path resource [org/springframework/cloud/netflix/zuul/ZuulProxyAutoConfiguration$EndpointConfiguration.class] and overriding is disabled.

Action:

Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true

Process finished with exit code 1
```
**进过排查之后发现是springboot版本高，我的是2.1.1版本，springcloud版本是F版本，只要把springboot版本调到2.0.6就行了（在父model里改）**

再次访问http://localhost:8766/api-a/hi?name=lisi  
![](https://upload-images.jianshu.io/upload_images/13532499-06f08e0f1effd8f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
访问http://localhost:8766/api-b/hi?name=lisi
![](https://upload-images.jianshu.io/upload_images/13532499-b05721abd746bebf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
证明了zuul服务有路由作用

## 四、服务过滤

zuul也可以有过滤作用，可以用来当安全验证
在ServerZuulApplication所在目录下新建一个MyFilter类

```
package com.example.serverzuul;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;

@Component
public class MyFilter extends ZuulFilter {

    private static Logger log = LoggerFactory.getLogger(MyFilter.class);
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        log.info(String.format("%s >>> %s", request.getMethod(), request.getRequestURL().toString()));
        Object accessToken = request.getParameter("token");
        if(accessToken == null) {
            log.warn("token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            try {
                ctx.getResponse().getWriter().write("token is empty");
            }catch (Exception e){}

            return null;
        }
        log.info("ok");
        return null;
    }
}
```
- filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下：                                
     ​    - pre：路由之前
     ​    - routing：路由之时
     ​    - post： 路由之后
     ​    - error：发送错误调用
- filterOrder：过滤的顺序
- shouldFilter：这里可以写逻辑判断，是否要过滤，本文true,永远过滤。
- run：过滤器的具体逻辑。可用很复杂，包括查sql，nosql去判断该请求到底有没有权限访问。
  访问：[http://localhost:8766/api-a/hi?name=lisi](http://localhost:8766/api-a/hi?name=lisi) 
  ![](https://upload-images.jianshu.io/upload_images/13532499-421964dba6230174.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  控制台
  ![](https://upload-images.jianshu.io/upload_images/13532499-c53dc465045781b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  访问：[http://localhost:8766/api-a/hi?name=lisi](http://localhost:8766/api-a/hi?name=lisi)
  ![](https://upload-images.jianshu.io/upload_images/13532499-1f0ee01defe5fef3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  控制台
  ![](https://upload-images.jianshu.io/upload_images/13532499-a76ee0f7de263e59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ## 参考资料

  大佬的文章 https://blog.csdn.net/forezp/article/details/81041012