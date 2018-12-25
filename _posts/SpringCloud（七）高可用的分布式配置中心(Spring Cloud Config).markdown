> 上篇文章讲述了一个服务如何从配置中心读取文件，配置中心如何从远程git读取配置文件，当服务实例很多时，都从配置中心读取文件，这时可以考虑将配置中心做成一个微服务，将其集群化，从而达到高可用，架构图如下：
> ![](https://upload-images.jianshu.io/upload_images/13532499-337e71e3f8e6fa56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 一，准备工作

继续使用上一篇文章的工程，创建一个eureka-server工程，用作服务注册中心。

在其pom.xml文件引入Eureka的起步依赖spring-cloud-starter-netflix- eureka-server，代码如下:
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>demo2-springcloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>eureka-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>eureka-server</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```
application.yml代码
```
server:
  port: 8889

eureka:
  instance:
    hostname: localhost
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```
在启动类添加@EnableEurekaServer注解
```
package com.example.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```
## 二，改造config-server

将其注册微到服务注册中心，作为Eureka客户端,pom添加eureka依赖

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```
修改application.yml配置文件
```
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/forezp/SpringcloudConfig/
          search-paths: respo
          password:
          username:
      label: master
server:
  port: 8888
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8889/eureka/
```
在启动类添加注解
```
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@EnableEurekaClient
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```
## 三，改造config-client

将其注册微到服务注册中心，作为Eureka客户端,pom添加依赖

```
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```
在启动类添加@EnableEurekaClient注解
```
package com.example.configclient;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@EnableEurekaClient
@RestController
@SpringBootApplication
public class ConfigClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }

    @Value("${democonfigclient.message}")
    String message;
    @RequestMapping("/hi")
    public String sayHi(){
        return message;
    }
}
```
配置文件bootstrap.properties，注意是bootstrap!修改bootstrap.yml
```
spring:
  application:
    name: config-client
  cloud:
    config:
      label: master
      profile: dev
      uri: http://localhost:8888/eureka/
      name: config-client
      discovery:
        enabled: true
        service-id: config-server
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8889/eureka/
server:
  port: 8881
```
- spring.cloud.config.discovery.enabled 是从配置中心读取文件。
- spring.cloud.config.discovery.serviceId 配置中心的servieId，即服务名。

这时发现，在读取配置文件不再写ip地址，而是服务名，这时如果配置服务部署多份，通过负载均衡，从而高可用。
依次启动eureka-servr,config-server,config-client
访问网址：[http://localhost:8889/](http://localhost:8889/)
![](https://upload-images.jianshu.io/upload_images/13532499-b631284e4382a48b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
访问http://localhost:8881/hi，浏览器显示：
![](https://upload-images.jianshu.io/upload_images/13532499-6da0048266d69002.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 四，易错总结

1.config-client的配置文件修改是在bootstrap.yml上
2.注册服务中心的路径不能不一样，不小心打错了，排查了几个小时。。。。
3.启动类添加的是@EnableEurekaClient不是@EnableEurekaServer

## 五，参考资料

大佬的文章https://blog.csdn.net/forezp/article/details/70037513