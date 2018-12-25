## 前言

在微服务架构中，根据业务来拆分成一个个的服务，服务与服务之间可以相互调用（RPC），在Spring Cloud可以用RestTemplate+Ribbon和Feign来调用。为了保证其高可用，单个服务通常会集群部署。由于网络原因或者自身的原因，服务并不能保证100%可用，如果单个服务出现问题，调用这个服务就会出现线程阻塞，此时若有大量的请求涌入，Servlet容器的线程资源会被消耗完毕，导致服务瘫痪。服务与服务之间的依赖性，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的“雪崩”效应。

为了解决这个问题，业界提出了断路器模型。

## 一、断路器简介

在一个分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，如何能够保证在一个依赖出问题的情况下，不会导致整体服务失败，这个就是Hystrix需要做的事情。Hystrix提供了熔断、隔离、Fallback、cache、监控等功能，能够在一个、或多个依赖同时出现问题时保证系统依然可用。

Netflix开源了Hystrix组件，实现了断路器模式，SpringCloud对这一组件进行了整合。 在微服务架构中，一个请求需要调用多个服务是非常常见的，如下图：
![](https://upload-images.jianshu.io/upload_images/13532499-197ed3c78f0c2618.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
较底层的服务如果出现故障，会导致连锁故障。当对特定的服务的调用的不可用达到一个阀值（Hystric 是5秒20次） 断路器将会被打开。
![](https://upload-images.jianshu.io/upload_images/13532499-8fdf34889a8e6bf8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
断路打开后，可用避免连锁故障，fallback方法可以直接返回一个固定值。

## 二、准备工作

这篇文章基于上一篇文章的工程，首先启动上一篇文章的工程，启动eureka-server 工程；启动eureka-client工程，它的端口为8762。

## 三、在Ribbon中使用路断器

改造serice-ribbon 工程的代码，首先在pox.xml文件中加入spring-cloud-starter-netflix-hystrix的起步依赖：

```
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```
在程序的启动类ServiceRibbonApplication 加@EnableHystrix注解开启Hystrix：
```
package com.example.serverribbon;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@EnableHystrix
@EnableEurekaClient
@EnableDiscoveryClient
@SpringBootApplication
public class ServerRibbonApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServerRibbonApplication.class, args);
    }

    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```
改造HelloService类，在HelloService方法上加上@HystrixCommand注解。该注解对该方法创建了熔断器的功能，并指定了fallbackMethod熔断方法，熔断方法直接返回了一个字符串，字符串为"hi,"+name+",sorry,error!"，代码如下:
```
package com.example.serverribbon.service;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class HelloService {

    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "hierror")
    public String hiService(String name) {
        return restTemplate.getForObject("http://eureka-client/hi?name="+name,String.class);
    }


    public String hierror(String name){ return "hi,"+name+",sorry,error!";}

}
```
启动：service-ribbon 工程，当我们访问http://localhost:8764/hi?name=lisi,浏览器显示：
![](https://upload-images.jianshu.io/upload_images/13532499-c160317dcff8878e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时关闭eureka-client工程，当我们再访问http://localhost:8764/hi?name=lisi，浏览器会显示：
![](https://upload-images.jianshu.io/upload_images/13532499-7216a4dde02a530d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这就说明当eureka-client工程不可用的时候，service-ribbon调用eureka-client的API接口时，会执行快速失败，直接返回一组字符串，而不是等待响应超时，这很好的控制了容器的线程阻塞。

## 四、Feign中使用路断器

Feign是自带断路器的，在D版本的Spring Cloud之后，它没有默认打开。需要在配置文件中配置打开它，在配置文件加以下代码：

```
feign:
  hystrix:
    enabled: true
```
基于service-feign工程进行改造，只需要在FeignClient的HiService接口的注解中加上fallback的指定类就行了：
```
package com.example.servicefeign.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(value = "eureka-client",fallback = ServiceHiHystrix.class)
public interface HiService {
    @RequestMapping(value = "hi",method = RequestMethod.GET)
    String sayhiclient(@RequestParam(value = "name") String name);
}
```
ServiceHiHystric需要实现HiService接口，并注入到Ioc容器中，代码如下：
```
package com.example.servicefeign.service;

import org.springframework.stereotype.Component;

@Component
public class ServiceHiHystrix implements HiService {
    @Override
    public String sayhiclient(String name) {
        return "sorry"+name;
    }
}
```
启动servcie-feign工程，浏览器打开http://localhost:8765/hi?name=lisi,注意此时eureka-client工程没有启动，网页显示：
![](https://upload-images.jianshu.io/upload_images/13532499-5a22d80791e09c48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
再次启动eureka-client工程后，用浏览器打开http://localhost:8765/hi?name=lisi
![](https://upload-images.jianshu.io/upload_images/13532499-8d4a629617ab07de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考资料

大佬的文章  https://blog.csdn.net/forezp/article/details/81040990