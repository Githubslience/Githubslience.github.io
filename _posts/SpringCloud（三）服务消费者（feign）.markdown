## 一、Feign简介

Feign是一个声明式的伪Http客户端，它使得写Http客户端变得更简单。使用Feign，只需要创建一个接口并注解。它具有可插拔的注解特性，可使用Feign 注解和JAX-RS注解。Feign支持可插拔的编码器和解码器。Feign默认集成了Ribbon，并和Eureka结合，默认实现了负载均衡的效果。

简而言之：

- Feign 采用的是基于接口的注解
- Feign 整合了ribbon，具有负载均衡的能力
- 整合了Hystrix，具有熔断的能力

## 二、准备工作

继续用上一节的工程， 启动eureka-server，端口为8761; 启动eureka-client 两次，端口分别为8762 、8763.

## 三、创建feign服务

新建一个spring-boot工程，取名为serice-feign，在它的pom文件引入Feign的起步依赖spring-cloud-starter-feign、Eureka的起步依赖spring-cloud-starter-netflix-eureka-client、Web的起步依赖spring-boot-starter-web，代码如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>service-feign</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>service-feign</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
</project>
```
在工程的配置文件application.yml文件，指定程序名为service-feign，端口号为8765，服务注册地址为http://localhost:8761/eureka/ ，代码如下：
```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

server:
  port: 8765

spring:
  application:
    name: service-feign
```
在程序的启动类ServiceFeignApplication ，加上@EnableFeignClients注解开启Feign的功能：
```
package com.example.servicefeign;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableEurekaClient
@EnableDiscoveryClient
@EnableFeignClients
@SpringBootApplication
public class ServiceFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceFeignApplication.class, args);
    }
}
```
定义一个feign接口，通过@ FeignClient（“服务名”），来指定调用哪个服务。比如在代码中调用了eureka-client服务的“/hi”接口，代码如下：
```
package com.example.servicefeign.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(value = "eureka-client")
public interface HiService {
    @RequestMapping(value = "hi",method = RequestMethod.GET)
    String sayhiclient(@RequestParam(value = "name") String name);
}
```
在Web层的controller层，对外暴露一个"/hi"的API接口，通过上面定义的Feign客户端HiService来消费服务。代码如下：
```
package com.example.servicefeign.controller;

import com.example.servicefeign.service.HiService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HiController {

    //编译器报错，无视。 因为这个Bean是在程序启动的时候注入的，编译器感知不到，所以报错。
    @Autowired
    HiService service;

    @GetMapping(value = "/hi")
    public String sayHi(@RequestParam String name) {
        return service.sayhiclient( name );
    }

}
```
启动程序，多次访问 http://localhost:8765/hi?name=forezp,浏览器交替显示：
![8673](https://upload-images.jianshu.io/upload_images/13532499-6e4c5c6b36e30e60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![8672](https://upload-images.jianshu.io/upload_images/13532499-afa8e2bdd63be9e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Feign源码解析：[http://blog.csdn.net/forezp/article/details/73480304](http://blog.csdn.net/forezp/article/details/73480304)

## 参考资料

大佬的文章 https://blog.csdn.net/forezp/article/details/81040965