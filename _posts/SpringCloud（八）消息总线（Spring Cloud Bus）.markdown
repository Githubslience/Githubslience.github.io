> Spring Cloud Bus 将分布式的节点用轻量的消息代理连接起来。它可以用于广播配置文件的更改或者服务之间的通讯，也可以用于监控。本文要讲述的是用Spring Cloud Bus实现通知微服务架构的配置文件的更改。

## 一，准备工作

续用上次工程，根据文档添加spring-cloud-starter-bus-amqp依赖，下载RabbitMQ，下载前了解RabbitMQ和erlang

## 二，改造config-client

在pom上添加spring-cloud-starter-bus-amqp和spring-boot-starter-actuator依赖

```
<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>
```
配置文件添加rabbitMq配置，包括RabbitMq的地址、端口，用户名、密码。并需要加上spring.cloud.bus的三个配置，具体如下：
```
spring:
  application:
    name: config-client
  cloud:
    config:
      label: master
      profile: dev
      uri: http://localhost:8888/eureka/
      discovery:
        enabled: true
        service-id: config-server
    bus:
      trace:
        enabled: true
      enabled: true
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8889/eureka/
server:
  port: 8881
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```
启动类添加@RefreshScope注解
```
package com.example.configclient;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RefreshScope
@EnableEurekaClient
@RestController
@SpringBootApplication
public class ConfigClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }

    @Value("${foo}")
    String message;
    @RequestMapping("/hi")
    public String sayHi(){
        return message;
    }

}
```
依次启动eureka-server，config-server，config-client
访问[http://localhost:8881/hi](http://localhost:8881/hi)
![](https://upload-images.jianshu.io/upload_images/13532499-1342a87d50d06c07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这时我们去代码仓库将foo的值改为“foo version 5”，即改变配置文件foo的值。如果是传统的做法，需要重启服务，才能达到配置文件的更新。此时，我们只需要发送post请求：[http://localhost:8881/actuator/bus-refresh](http://localhost:8881/actuator/bus-refresh)
再次访问[http://localhost:8881/hi](http://localhost:8881/hi)
![](https://upload-images.jianshu.io/upload_images/13532499-57431e2cc2f6d603.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 三，分析

此时的架构图为
![](https://upload-images.jianshu.io/upload_images/13532499-389ba548bbcd18e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当git文件更改的时候，通过pc端用post 向端口为8882的config-client发送请求/bus/refresh／；此时8882端口会发送一个消息，由消息总线向其他服务传递，从而使整个微服务集群都达到更新配置文件

## 四，参考资料

https://blog.csdn.net/forezp/article/details/81041062