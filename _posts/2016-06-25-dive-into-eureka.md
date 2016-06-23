---
layout:     post
title:      Dive into Eureka
date:       2016-06-20 20:00:00 +0800
summary:    本文主要介绍了Eureka的实现细节
categories:
---

由于项目中使用了Eureka做服务注册和发现，所以最近花了一些时间比较深入的研究了一下<a href="https://github.com/Netflix/eureka" target="_blank">Eureka</a>，今天就来介绍一下它的实现细节。

## 1. What is Eureka?
其官方文档中对自己的定义是：

> Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers. We call this service, the Eureka Server. Eureka also comes with a Java-based client component,the Eureka Client, which makes interactions with the service much easier. The client also has a built-in load balancer that does basic round-robin load balancing.

简单来说Eureka就是Netflix开源的一款提供服务注册和发现的产品，并且提供了相应的Java客户端。

## 2. Why Eureka?
那么为什么我们在项目中使用了Eureka呢？我大致总结了一下，有以下几方面的原因：

* **它提供了完整的Service Registry和Service Discovery实现**
	* 首先是提供了完整的实现，并且也经受住了Netflix自己的生产环境考验，相对使用起来会比较省心。

* **和Spring Cloud无缝集成**
	* 我们的项目本身就使用了Spring Cloud和Spring Boot，同时Spring Cloud还有一套非常完善的开源代码来整合Eureka，所以使用起来非常方便。
	* 另外，Eureka还支持在我们应用自身的容器中启动，也就是说我们的应用启动完之后，既充当了Eureka的角色，同时也是服务的提供者。这样就极大的提高了服务的可用性。
	
* **Open Source**
	* 最后一点是开源，由于代码是开源的，所以非常便于我们了解它的实现原理和排查问题。
	
## 3. Dive into Eureka
相信大家看到这里，已经对Eureka有了一个初步的认识，接下来我们就来深入了解下它吧~

### 3.1 Overview

#### 3.1.1 Basic Architecture

![architecture-overview](/images/2016-06-25/architecture-overview.png)

上图简要描述了Eureka的基本架构，也就是由3个角色组成：

1. Eureka Server
	* 提供服务注册和发现
2. Service Provider
	* 服务提供方
	* 将自身服务注册到Eureka，从而是服务消费方能够找到
3. Service Consumer
	* 服务消费方
	* 从Eureka获取注册服务列表，从而能够消费服务

需要注意的是，上图中的3个角色都是逻辑角色。在实际运行中，这几个角色甚至可以是同一个实例，比如在我们项目中，Eureka Server和Service Provider就是同一个JVM进程。

#### 3.1.2 More in depth

![architecture-detail](/images/2016-06-25/architecture-detail.png)

上图更进一步的展示了3个角色之间的交互。

1. Service Provider会向Eureka Server做Register（服务注册）、Renew（服务续约）、Cancel（服务下线）等操作。
2. Eureka Server之间会做注册服务的同步
3. Service Consumer会向Eureka Server获取注册服务列表，并消费服务

### 3.2 Demo

为了给大家一个更直观的印象，我们可以通过一个简单的demo来实际运行一下，从而对Eureka有更好的了解。

#### 3.2.1 Git Repository

Git仓库：git@github.com:nobodyiam/spring-cloud-in-action.git

这个项目使用了Spring Cloud相关类库，包括：

* Spring Cloud Config
* Spring Cloud Eureka

#### 3.2.2 准备工作

由于项目使用了Spring Cloud Config做配置，所以第一步先在本地启动Config Server。

由于项目基于Spring Boot开发，所以直接运行`com.nobodyiam.spring.cloud.in.action.config.ConfigServerApplication`即可。

#### 3.2.3 Eureka Server Demo

Eureka Server的Demo模块名是：`eureka-server`。

#### 3.2.3.1 Maven依赖

`eureka-server`是一个基于Spring Boot的Web应用，我们首先需要做的就是在pom中引入Spring Cloud Eureka Server的依赖。
	
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
        <version>1.1.0.RC2</version>
    </dependency>

#### 3.2.3.2 启用Eureka Server

启用Eureka Server非常简单，只需要加上`@EnableEurekaServer`即可。

{% highlight java%}
@EnableEurekaServer
@SpringBootApplication
public class EurekaServiceApplication {
  public static void main(String[] args) {
    SpringApplication.run(EurekaServiceApplication.class, args);
  }
}
{% endhighlight %}

做完以上配置后，启动应用，Eureka Server就开始工作了！

启动完，打开http://localhost:8761就能看到启动成功的画面了。

![eureka-server-init-page](/images/2016-06-25/eureka-server-init-page.png)

#### 3.2.4 Service Provider and Service Consumer Demo

Service Provider的Demo模块名是：`reservation-service`。

Service Consumer的Demo模块名是：`reservation-client`。

#### 3.2.4.1 Maven依赖

`reservation-service`和`reservation-client`都是基于Spring Boot的Web应用，我们首先需要做的就是在pom中引入Spring Cloud Eureka的依赖。

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
        <version>1.1.0.RC2</version>
    </dependency>

#### 3.2.4.2 启动Service Provider
启用Service Provider非常简单，只需要加上`@EnableDiscoveryClient`即可。

{% highlight java%}
@EnableDiscoveryClient
@SpringBootApplication
public class ReservationServiceApplication {
  public static void main(String[] args) {
    new SpringApplicationBuilder(ReservationServiceApplication.class)
            .run(args);
  }
}
{% endhighlight %}

做完以上配置后，启动应用，Server Provider就开始工作了！

启动完，打开http://localhost:8761就能看到服务已经注册到Eureka Server了。

![eureka-server-with-service-provider](/images/2016-06-25/eureka-server-with-service-provider.png)

#### 3.2.4.3 启动Service Consumer

启动Service Consumer其实和Service Provider一样，因为本质上Eureka提供的客户端是不区分Provider和Consumer的，一般情况下，Provider同时也会是Consumer。

{% highlight java%}
@EnableDiscoveryClient
@SpringBootApplication
public class ReservationClientApplication {
  @Bean
  CommandLineRunner runner(DiscoveryClient dc) {
    return args -> {
      dc.getInstances("reservation-service")
              .forEach(si -> System.out.println(String.format(
                      "Found %s %s:%s", si.getServiceId(), si.getHost(), si.getPort())));
    };
  }
  public static void main(String[] args) {
    SpringApplication.run(ReservationClientApplication.class, args);
  }
}
{% endhighlight %}

上述代码中通过`dc.getInstances("reservation-service")`就能获取到当前所有注册的`reservation-service`服务。

### 3.3 Eureka Server实现细节

看了前面的demo，我们已经初步领略到了Spring Cloud和Eureka的强大之处，通过短短几行配置就实现了服务注册和发现！

相信大家一定想了解Eureka是如何实现的吧，所以接下来我们继续Dive！

### 3.4 Service Provider实现细节


### 3.5 Service Consumer实现细节

## 4. Summary
