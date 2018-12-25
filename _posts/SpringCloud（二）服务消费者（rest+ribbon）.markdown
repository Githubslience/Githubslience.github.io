> 在上一篇文章，讲了服务的注册和发现。在微服务架构中，业务都会被拆分成一个独立的服务，服务与服务的通讯是基于http restful的。Spring cloud有两种服务调用方式，一种是ribbon+restTemplate，另一种是feign。在这一篇文章首先讲解下基于ribbon+rest。

## 一，ribbon简介

Spring Cloud Ribbon是一个基于HTTP和TCP的客户端负载均衡工具，它基于Netflix Ribbon实现。通过Spring Cloud的封装，可以让我们轻松地将面向服务的REST模版请求自动转换成客户端负载均衡的服务调用。Spring Cloud Ribbon虽然只是一个工具类框架，它不像服务注册中心、配置中心、API网关那样需要独立部署，但是它几乎存在于每一个Spring Cloud构建的微服务和基础设施中。因为微服务间的调用，API网关的请求转发等内容，实际上都是通过Ribbon来实现的，包括后续我们将要介绍的Feign，它也是基于Ribbon实现的工具。所以，对Spring Cloud Ribbon的理解和使用，对于我们使用Spring Cloud来构建微服务非常重要。

## 二、准备工作

这一篇文章基于上一篇文章的工程，启动eureka-server 工程；启动eureka-client工程，它的端口为8762；将service-hi的配置文件的端口改为8763,并启动，这时你会发现：eureka-client在eureka-server注册了2个实例，这就相当于一个小的集群。

## 三、建立一个服务消费者

重新新建一个spring-boot工程，取名为：service-ribbon;
在它的pom.xml继承了父pom文件，并引入了以下依赖：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>server-ribbon</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>server-ribbon</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

</project>
```
在工程的配置文件指定服务的注册中心地址为http://localhost:8761/eureka/，程序名称为 service-ribbon，程序端口为8764。配置文件application.yml如下：
```
server:
  port: 8764

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: service-ribbon
```
在工程的启动类中,通过@EnableDiscoveryClient向服务中心注册；并且向程序的ioc注入一个bean: restTemplate;并通过@LoadBalanced注解表明这个restRemplate开启负载均衡的功能。
![](https://upload-images.jianshu.io/upload_images/13532499-82b78e3042466123.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
写一个测试类HelloService，通过之前注入ioc容器的restTemplate来消费eureka-client服务的“/hi”接口，在这里我们直接用的程序名替代了具体的url地址，在ribbon中它会根据服务名来选择具体的服务实例，根据服务实例在请求的时候会用具体的url替换掉服务名，代码如下：
```
package com.example.serverribbon.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class HelloService {

    @Autowired
    RestTemplate restTemplate;

    public String hiService(String name) {
        return restTemplate.getForObject("http://eureka-client/hi?name="+name,String.class);
    }
}
```
写一个controller，在controller中用调用HelloService 的方法，代码如下：
```
package com.example.serverribbon.controller;

import com.example.serverribbon.service.HelloService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    @Autowired
    HelloService helloService;

    @GetMapping(value = "/hi")
    public String hi(@RequestParam String name) {
        return helloService.hiService( name );
    }
}
```
在浏览器上多次访问http://localhost:8764/hi?name=lisi，浏览器交替显示：
![8762](https://upload-images.jianshu.io/upload_images/13532499-eff97bcd960c4162.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![8763](https://upload-images.jianshu.io/upload_images/13532499-0cbf4be45b684c04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这说明当我们通过调用restTemplate.getForObject(“[http://SERVICE-HI/hi?name=](http://service-hi/hi?name=)”+name,String.class)方法时，已经做了负载均衡，访问了不同的端口的服务实例。

## 四、架构

![架构](https://upload-images.jianshu.io/upload_images/13532499-41efcbd5ab78f490.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 一个服务注册中心，eureka server,端口为8761
- service-hi工程跑了两个实例，端口分别为8762,8763，分别向服务注册中心注册
- sercvice-ribbon端口为8764,向服务注册中心注册
- 当sercvice-ribbon通过restTemplate调用service-hi的hi接口时，因为用ribbon进行了负载均衡，会轮流的调用---- service-hi：8762和8763 两个端口的hi接口；

## 参考资料

大佬的原文：https://blog.csdn.net/forezp/article/details/81040946 