## 一、Spring Cloud是什么

Spring Cloud 是一系列框架的有序集合，它利用 Spring Boot 的开发便利性简化了分布式系统的开发，比如服务发现、服务网关、服务路由、链路追踪等。Spring Cloud 并不重复造轮子，而是将市面上开发得比较好的模块集成进去，进行封装，从而减少了各模块的开发成本。换句话说：Spring Cloud 提供了构建分布式系统所需的“全家桶”。

## 二、创建服务注册中心

### 2.1 首先创建一个主Maven工程，在其pom文件引入依赖

spring Boot版本为2.0.3.RELEASE，Spring Cloud版本为Finchley.RELEASE。这个pom文件作为父pom文件，起到依赖版本控制的作用，其他module工程继承该pom。这一系列文章全部采用这种模式，其他文章的pom跟这个pom一样。再次说明一下，以后不再重复引入。代码如下：
用的是多module结构

父module的pom代码
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>spring-cloud</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>spring-cloud</name>
    <description>spring-cloud project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <modules>
        <module>eureka-server</module>
    </modules>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

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
### 2.2 创建子module

右键工程->创建module-> 选择spring initialir 如下图：
![](https://upload-images.jianshu.io/upload_images/13532499-dda355b0698eeb38.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/13532499-a452676fb39059a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下一步->选择cloud discovery->eureka server ,然后一直下一步就行了。
![](https://upload-images.jianshu.io/upload_images/13532499-1076b5145794e34d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
创建完后的工程，其pom.xml继承了父pom文件，并引入spring-cloud-starter-netflix-eureka-server的依赖，代码如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>eureka-server</name>
    <description>eureka-server project for Spring Boot</description>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>

</project>
```
** 启动一个服务注册中心**，只需要一个注解@EnableEurekaServer，这个注解需要在springboot工程的启动application类上加：
![](https://upload-images.jianshu.io/upload_images/13532499-d21304b4c476c2e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
eureka是一个高可用的组件，它没有后端缓存，每一个实例注册之后需要向注册中心发送心跳（因此可以在内存中完成），在默认情况下erureka server也是一个eureka client ,必须要指定一个 server。eureka server的配置文件appication.yml：
```
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
      
spring:
  application:
    name: eurka-server
```
通过eureka.client.registerWithEureka：false和fetchRegistry：false来表明自己是一个eureka server.
 eureka server 是有界面的，启动工程,打开浏览器访问：**http://localhost:8761**

### 2.3 创建一个服务提供者 (eureka client)

当client向server注册时，它会提供一些元数据，例如主机和端口，URL，主页等。Eureka server 从每个client实例接收心跳消息。 如果心跳超时，则通常将该实例从注册server中删除。

创建过程同server类似,创建完pom.xml如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>eureka-client</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>eureka-client</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
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
通过注解@EnableEurekaClient 表明自己是一个eurekaclient.
![](https://upload-images.jianshu.io/upload_images/13532499-dc1ae6f5ed132d0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
仅仅@EnableEurekaClient是不够的，还需要在配置文件中注明自己的服务注册中心的地址，application.yml配置文件如下：
```
server:
  port: 8762

spring:
  application:
    name: eureka-client

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```
需要指明spring.application.name,这个很重要，这在以后的服务与服务之间相互调用一般都是根据这个name 。
启动工程，打开http://localhost:8761 ，即eureka server 的网址：
![](https://upload-images.jianshu.io/upload_images/13532499-b0cab7a4898158b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
你会发现一个服务已经注册在服务中了，服务名为EUREKA-CLIENT,端口为8762
这时打开 [http://localhost:8762/hi?name=lisi](http://localhost:8762/hi?name=forezp) ，你会在浏览器上看到 :
![](https://upload-images.jianshu.io/upload_images/13532499-241e1a79a7b80560.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考资料

参考大佬的文章 https://blog.csdn.net/forezp/article/details/81040925