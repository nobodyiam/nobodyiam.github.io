---
layout:     post
title:      Dive into Eureka
date:       2016-06-25 20:00:00 +0800
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

上图简要描述了Eureka的基本架构，由3个角色组成：

1. **Eureka Server**
	* 提供服务注册和发现

2. **Service Provider**
	* 服务提供方
	* 将自身服务注册到Eureka，从而使服务消费方能够找到

3. **Service Consumer**
	* 服务消费方
	* 从Eureka获取注册服务列表，从而能够消费服务

需要注意的是，上图中的3个角色都是逻辑角色。在实际运行中，这几个角色甚至可以是同一个实例，比如在我们项目中，Eureka Server和Service Provider就是同一个JVM进程。

#### 3.1.2 More in depth

![architecture-detail](/images/2016-06-25/architecture-detail.png)

上图更进一步的展示了3个角色之间的交互。

1. Service Provider会向Eureka Server做Register（服务注册）、Renew（服务续约）、Cancel（服务下线）等操作。
2. Eureka Server之间会做注册服务的同步，从而保证状态一致
3. Service Consumer会向Eureka Server获取注册服务列表，并消费服务

### 3.2 Demo

为了给大家一个更直观的印象，我们可以通过一个简单的demo来实际运行一下，从而对Eureka有更好的了解。

#### 3.2.1 Git Repository

Git仓库：<a href="https://github.com/nobodyiam/spring-cloud-in-action" target="_blank">git@github.com:nobodyiam/spring-cloud-in-action.git</a>

这个项目使用了Spring Cloud相关类库，包括：

* <a href="https://cloud.spring.io/spring-cloud-config/" target="_blank">Spring Cloud Config</a>
* <a href="https://cloud.spring.io/spring-cloud-netflix/" target="_blank">Spring Cloud Eureka (Netflix)</a>

#### 3.2.2 准备工作

Demo项目使用了Spring Cloud Config做配置，所以第一步先在本地启动Config Server。

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

相信大家一定想了解Eureka是如何实现的吧，所以接下来我们继续Dive！首先来看下Eureka Server的几个对外接口实现。

#### 3.3.1 Register

首先来看Register（服务注册），这个接口会在Service Provider启动时被调用来实现服务注册。同时，当Service Provider的服务状态发生变化时（如自身检测认为Down的时候），也会调用来更新服务状态。

接口实现比较简单，如下图所示。

1. `ApplicationResource`类接收Http服务请求，调用`PeerAwareInstanceRegistryImpl`的`register`方法
2. `PeerAwareInstanceRegistryImpl`完成服务注册后，调用`replicateToPeers`向其它Eureka Server节点（Peer）做状态同步

![eureka-server-register](/images/2016-06-25/eureka-server-register.png)

注册的服务列表保存在一个嵌套的hash map中：

* 第一层hash map的key是app name，也就是应用名字
* 第二层hash map的key是instance name，也就是实例名字

以[3.2.4.2](#service-provider)中的截图为例，`RESERVATION-SERVICE`就是app name，`jason-mbp.lan:reservation-service:8000`就是instance name。

Hash map定义如下：

{% highlight java %}
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry =
new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
{% endhighlight %}

#### 3.3.2 Renew

Renew（服务续约）操作由Service Provider定期调用，类似于heartbeat。主要是用来告诉Eureka Server Service Provider还活着，避免服务被剔除掉。接口实现如下图所示。

可以看到，接口实现方式和`register`基本一致：首先更新自身状态，再同步到其它Peer。

![eureka-server-renew](/images/2016-06-25/eureka-server-renew.png)

#### 3.3.3 Cancel

Cancel（服务下线）一般在Service Provider shut down的时候调用，用来把自身的服务从Eureka Server中删除，以防客户端调用不存在的服务。接口实现如下图所示。

![eureka-server-cancel](/images/2016-06-25/eureka-server-cancel.png)

#### 3.3.4 Fetch Registries

Fetch Registries由Service Consumer调用，用来获取Eureka Server上注册的服务。

为了提高性能，服务列表在Eureka Server会缓存一份，同时每30秒更新一次。

![eureka-server-fetch](/images/2016-06-25/eureka-server-fetch.png)

#### 3.3.5 Eviction

Eviction（失效服务剔除）用来定期在Eureka Server检测失效的服务，检测标准就是超过一定时间没有Renew的服务。

默认失效时间为90秒，也就是如果有服务超过90秒没有向Eureka Server发起Renew请求的话，就会被当做失效服务剔除掉。

失效时间可以通过`eureka.instance.leaseExpirationDurationInSeconds`进行配置，定期扫描时间可以通过`eureka.server.evictionIntervalTimerInMs`进行配置。

接口实现逻辑见下图：

![eureka-server-evict](/images/2016-06-25/eureka-server-evict.png)

#### 3.3.6 How Peer Replicates

在前面的Register、Renew、Cancel接口实现中，我们看到了都会有replicateToPeers操作，这个就是用来做Peer之间的状态同步。

通过这种方式，Service Provider只需要通知到任意一个Eureka Server后就能保证状态会在所有的Eureka Server中得到更新。

具体实现方式其实很简单，就是接收到Service Provider请求的Eureka Server，把请求再次转发到其它的Eureka Server，调用同样的接口，传入同样的参数，除了会在header中标记isReplication=true，从而避免重复的replicate。

#### 3.3.7 How Peer Nodes are Discovered

那大家可能会有疑问，Eureka Server是怎么知道有多少Peer的呢？

Eureka Server在启动后会调用`EurekaClientConfig.getEurekaServerServiceUrls`来获取所有的Peer节点，并且会定期更新。定期更新频率可以通过`eureka.server.peerEurekaNodesUpdateIntervalMs`配置。

这个方法的默认实现是从配置文件读取，所以如果Eureka Server节点相对固定的话，可以通过在配置文件中配置来实现。

如果希望能更灵活的控制Eureka Server节点，比如动态扩容/缩容，那么可以override `getEurekaServerServiceUrls`方法，提供自己的实现，比如我们的项目中会通过数据库读取Eureka Server列表。

具体实现如下图所示：

![eureka-server-peer-discovery](/images/2016-06-25/eureka-server-peer-discovery.png)

#### 3.3.8 How New Peer Initializes

最后再来看一下一个新的Eureka Server节点加进来，或者Eureka Server重启后，如何来做初始化，从而能够正常提供服务。

具体实现如下图所示，简而言之就是启动时把自己当做是Service Consumer从其它Peer Eureka获取所有服务的注册信息。然后对每个服务，在自己这里执行Register，isReplication=true，从而完成初始化。

![eureka-server-peer-init](/images/2016-06-25/eureka-server-peer-init.png)

### 3.4 Service Provider实现细节

现在来看下Service Provider的实现细节，主要就是Register、Renew、Cancel这3个操作。

#### 3.4.1 Register

Service Provider要对外提供服务，一个很重要的步骤就是把自己注册到Eureka Server上。

这部分的实现比较简单，只需要在启动时和实例状态变化时调用Eureka Server的接口注册即可。需要注意的是，需要确保配置`eureka.client.registerWithEureka`=true。

![service-provider-register](/images/2016-06-25/service-provider-register.png)

#### 3.4.2 Renew

Renew操作会在Service Provider端定期发起，用来通知Eureka Server自己还活着。
这里有两个比较重要的配置需要注意一下：

1. `eureka.instance.leaseRenewalIntervalInSeconds`
	
	Renew频率。默认是30秒，也就是每30秒会向Eureka Server发起Renew操作。
	
2. `eureka.instance.leaseExpirationDurationInSeconds`
	
	服务失效时间。默认是90秒，也就是如果Eureka Server在90秒内没有接收到来自Service Provider的Renew操作，就会把Service Provider剔除。

具体实现如下：

![service-provider-renew](/images/2016-06-25/service-provider-renew.png)

#### 3.4.3 Cancel

在Service Provider服务shut down的时候，需要及时通知Eureka Server把自己剔除，从而避免客户端调用已经下线的服务。

逻辑本身比较简单，通过对方法标记`@PreDestroy`，从而在服务shut down的时候会被触发。

![service-provider-cancel](/images/2016-06-25/service-provider-cancel.png)

#### 3.4.4 How Eureka Servers are Discovered

这里大家疑问又来了，Service Provider是怎么知道Eureka Server的地址呢？

其实这部分的主体逻辑和[3.3.7 How Peer Nodes are Discovered](#how-peer-nodes-are-discovered)几乎是一样的。

也是默认从配置文件读取，如果需要更灵活的控制，可以通过override `getEurekaServerServiceUrls`方法来提供自己的实现。定期更新频率可以通过`eureka.client.eurekaServiceUrlPollIntervalSeconds`配置。

![client-discover-eureka-server](/images/2016-06-25/client-discover-eureka-server.png)

### 3.5 Service Consumer实现细节

Service Consumer这块的实现相对就简单一些，因为它只涉及到从Eureka Server获取服务列表和更新服务列表。

#### 3.5.1 Fetch Service Registries

Service Consumer在启动时会从Eureka Server获取所有服务列表，并在本地缓存。需要注意的是，需要确保配置`eureka.client.shouldFetchRegistry`=true。

![service-consumer-fetch-registries](/images/2016-06-25/service-consumer-fetch-registries.png)

#### 3.5.2 Update Service Registries

由于在本地有一份缓存，所以需要定期更新，定期更新频率可以通过`eureka.client.registryFetchIntervalSeconds`配置。

![service-consumer-update-registries](/images/2016-06-25/service-consumer-update-registries.png)

#### 3.5.3 How Eureka Servers are Discovered

Service Consumer和Service Provider一样，也有一个如何知道Eureka Server地址的问题。

其实由于Service Consumer和Service Provider本质上使用的是同一个Eureka客户端，所以这部分逻辑是一样的，这里就不再赘述了。详细信息见[3.4.4节](#how-eureka-servers-are-discovered)。

## 4. Summary

本文主要介绍了Eureka的实现思路，通过深入了解Eureka Server、Service Provider、Service Consumer的实现，我们清晰地看到了服务注册、发现的一系列过程和实现方式。

相信对正在使用Eureka的同学会有一些帮助，同时希望对暂不使用Eureka的同学也能有一定的启发，毕竟服务注册、发现还是比较基础和通用的，了解了实现方式后，在使用上应该能更得心应手一些吧~