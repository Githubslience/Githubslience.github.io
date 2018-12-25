> 单实例的服务注册中心一旦崩溃的时候，所有依赖的服务都会GG，所以Spring Cloud就提供了注册中心高可用的方案

## 一，准备工作

按之前[Spring Cloud（一）创建服务注册中心](https://www.jianshu.com/p/7999a72860a9)的方式来搭建高可用的注册中心

## 二，eureka-server

主要是配置文件的改动，新创建application-peer1.yml和application-peer2.yml
![](https://upload-images.jianshu.io/upload_images/13532499-7bff83a2c2655b93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
application-peer1.yml

```
server:
  port: 8761
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2:8762/eureka/
```
application-peer2.yml
```
server:
  port: 8762

spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1:8761/eureka/
```
application.yml
```
spring:
  application:
    name: eureka-server
  profiles: peer1
```
## 三，service-hi

主要是配置文件的修改
application.yml

```
eureka:
  instance:
    hostname: service-hi
  client:
    serviceUrl:
      defaultZone: http://peer1:8761/eureka/
server:
  port: 8763
spring:
  application:
    name: service-hi
```
## 四，启动服务

在启动前，按照官方文档的指示，需要改变etc/hosts，linux系统通过vim /etc/hosts ,windows电脑，在c:/windows/systems/drivers/etc/hosts 修改。

```
  127.0.0.1  	peer1
  127.0.0.1  	peer2
```
![EurekaServerApplication1](https://upload-images.jianshu.io/upload_images/13532499-22f8f2021e3f193a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![EurekaServerApplication2](https://upload-images.jianshu.io/upload_images/13532499-612cdb92a286f7d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

依次启动EurekaServerApplication1，EurekaServerApplication2，service-hi
访问[http://localhost:8761/](http://localhost:8761/)
![8761](https://upload-images.jianshu.io/upload_images/13532499-b1a8ab29680ff039.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
访问[http://localhost:8762/](http://localhost:8762/)
![8762](https://upload-images.jianshu.io/upload_images/13532499-25950e2f5c4b4344.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
你会发现注册了service-hi，并且有个peer2节点，同理访问localhost:8769你会发现有个peer1节点。
client只向8761注册，但是你打开8769，你也会发现，8769也有 client的注册信息。
此时的架构图：
![](https://upload-images.jianshu.io/upload_images/13532499-49450221603facad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
peer1和peer2相互注册
Eureka-eserver peer1 8761,Eureka-eserver peer2 8769相互感应，当有服务注册时，两个Eureka-eserver是对等的，它们都存有相同的信息，这就是通过服务器的冗余来增加可靠性，当有一台服务器宕机了，服务并不会终止，因为另一台服务存有相同的数据。

## 五，参考资料

https://blog.csdn.net/forezp/article/details/81041101#commentBox