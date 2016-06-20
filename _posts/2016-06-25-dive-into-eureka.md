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

* 它提供了完成的Service Registry和Service Discovery实现
	* 首先是提供了完整的实现，并且也经受住了Netflix自己的生产环境考验，相对使用起来会比较省心。

* 和Spring Cloud无缝集成
	* 我们的项目本身就使用了Spring Cloud和Spring Boot，同时Spring Cloud还有一套非常完善的开源代码来整合Eureka，所以使用起来非常方便。
	* 另外，Eureka还支持在我们应用自身的容器中启动，也就是说我们的应用启动完之后，既充当了Eureka的角色，同时也是服务的提供者。这样就极大的提高了服务的可用性。
	
* Open Source
	* 最后一点是开源，由于代码是开源的，所以非常便于我们了解它的实现原理和排查问题。
	
## 3. Dive into Eureka

### 3.1 Overview


### 3.2 Demo


### 3.3 Eureka Server实现细节


### 3.4 Service Provider实现细节


### 3.5 Service Consumer实现细节

## 4. Summary
