---
layout:     post
title:      配置中心，让微服务更『智能』
date:       2018-07-29 13:00:00 +0800
summary:    随着微服务的流行，应用和机器数量急剧增长，程序配置也愈加繁杂：各种功能的开关、参数的配置、服务器的地址等等。同时，我们对程序配置的期望值也越来越高：配置修改后实时生效，灰度发布，分环境、分集群管理，完善的权限、审核机制等等。在这样的大环境下，传统的通过配置文件、数据库等方式已经越来越无法满足我们对配置管理的需求。
categories:
---
# 1. 背景介绍
随着微服务的流行，应用和机器数量急剧增长，程序配置也愈加繁杂：各种功能的开关、参数的配置、服务器的地址等等。
同时，我们对程序配置的期望值也越来越高：配置修改后实时生效，灰度发布，分环境、分集群管理，完善的权限、审核机制等等。

在这样的大环境下，传统的通过配置文件、数据库等方式已经越来越无法满足我们对配置管理的需求。

配置中心，应运而生！

通过配置中心，我们可以方便地管理微服务在不同环境中的配置，从而可以在运行时动态调整服务行为，真正实现配置即『控制』的目标。
所以，在一定程度上，配置中心就成为了微服务的大脑，如何用好这个大脑，让微服务更『智能』，也就成为了一项比较重要的议题。

# 2. 为什么需要配置中心？

## 2.1 配置即『控制』

程序的发布其实和卫星的发射有一些相似之处。

当卫星发射升天后，基本就处于自主驾驶状态了，一般情况下就是按照预设的轨道运行，间歇可能会收到一些来自地面的『控制』信号对运行姿态进行一定的调整。

![satellite-communication](/images/2018-07-29/satellite-communication.jpg)

*[图片来源](https://www.popularmechanics.com/space/a7194/how-it-works-nasas-experimental-laser-communication-system/)*

程序发布其实也是这样，当程序发布到生产环境后，一般就是按照预设的逻辑运行，我们无法直接去干预程序的行为，不过可以通过调整配置参数来动态调整程序的行为。这些配置参数就代表着我们对程序的『控制』信号。

由此可见，配置对程序的运行非常重要，我们需要一种可靠性高、实时性好的手段，从而可以随时对程序『发号施令』。

## 2.2 配置需要治理

鉴于配置对程序正确运行的重要性，所以配置的治理就显得尤为重要了，比如：

1. 权限控制、审计日志
  * 由于配置能改变程序的行为，不正确的配置甚至能引起灾难，所以对配置的修改必须有比较完善的权限控制。同时也需要有一套完善的审计机制，能够方便地追溯是谁改的配置、改了什么、什么时候改的等等。

2. 灰度发布、配置回滚
  * 对于一些比较重要的配置变更，我们一般会倾向于先在少量机器上修改看看效果，如果没问题再推给所有机器。同时如果发现配置改得有问题的话，需要能够方便地回滚配置。

3. 不同环境、集群管理
  * 同一份程序在不同的环境（开发，测试，生产）、不同的集群（如不同的数据中心）经常需要有不同的配置，所以需要有完善的环境、集群配置管理。

## 2.3 微服务的复杂性

单体应用时代，应用数量比较少，配置也相对比较简单，还有可能让运维登上机器一台一台修改程序的配置文件。

随着微服务的流行，大应用拆成小应用，小应用拆成多个独立的服务，导致微服务的节点数量非常多，配置也随着服务数量增加而急剧增长，再让运维登上机器一台一台手工修改配置不仅效率低，而且还容易出错。如果碰到了紧急事件需要大规模迅速修改配置，估计运维人员也只能两手一摊了。

![satellite-communication](/images/2018-07-29/microservices-architecture.png)

*[图片来源](https://www.nginx.com/blog/introduction-to-microservices/)*

所以，综合以上几个要素，我们需要一个统一的配置中心来管理微服务的配置。

# 3. 配置中心的一般模样

那么，我们应该需要一个什么样的配置中心呢？

接下来就以[开源配置中心Apollo](https://github.com/ctripcorp/apollo)为例，来看一下配置中心的一般模样。

## 3.1 治理能力

如前面所论述的：配置需要治理，所以配置中心需要具备完善的治理能力，比如：

1. 统一管理不同环境、不同集群的配置
2. 支持灰度发布
3. 支持已发布的配置回滚
4. 完善的权限管理、操作审计日志

Apollo配置中心的管理界面如下图所示，可以发现相应的治理功能还是非常齐全的。

![Apollo](/images/2016-07-09/portal-overview.png)

## 3.2 可用性

配置即『控制』，所以在一定程度上，配置中心已经成为了微服务的大脑，作为大脑，可用性显然是要求非常高的。

接下来我们一起看一下Apollo是怎么实现高可用的。

### 3.2.1 Apollo at a glance

如下即是Apollo的基础模型：

1. 用户在配置中心对配置进行修改并发布
2. 配置中心通知Apollo客户端有配置更新
3. Apollo客户端从配置中心拉取最新的配置、更新本地配置并通知到应用

![basic-architecture](/images/2016-07-09/basic-architecture.png)

### 3.2.2 服务端高可用

![overall-architecture](/images/2016-07-09/overall-architecture.png)

上图简要描述了Apollo的服务端设计，我们可以从下往上看：

* Config Service提供配置的读取、推送等功能，服务对象是Apollo客户端
* Admin Service提供配置的修改、发布等功能，服务对象是Apollo Portal（管理界面）
* Config Service和Admin Service都是多实例、无状态部署，所以需要将自己注册到Eureka中并保持心跳
* 在Eureka之上我们架了一层Meta Server用于封装Eureka的服务发现接口，主要是为了让客户端和Eureka解耦
* Client通过域名访问Meta Server获取Config Service服务列表（IP+Port），而后直接通过IP+Port访问服务，同时在Client侧会做load balance、错误重试
* Portal通过域名访问Meta Server获取Admin Service服务列表（IP+Port），而后直接通过IP+Port访问服务，同时在Portal侧会做load balance、错误重试
* 为了简化部署，我们实际上会把Config Service、Eureka和Meta Server三个逻辑角色部署在同一个JVM进程中
* 通过上述的设计，可以看到整个服务端是无单点，有效地保证了服务端的可用性

### 3.2.3 客户端高可用

![client-architecture](/images/2016-07-09/client-architecture.png)

上图简要描述了Apollo客户端的实现原理：

1. 客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。（通过Http Long Polling实现）
2. 客户端还会定时从Apollo配置中心服务端拉取应用的最新配置。
    * 这是一个fallback机制，为了防止推送机制失效导致配置不更新
    * 客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回304 - Not Modified
    * 定时频率默认为每5分钟拉取一次，客户端也可以通过在运行时指定System Property: `apollo.refreshInterval`来覆盖，单位为分钟。
3. 客户端从Apollo配置中心服务端获取到应用的最新配置后，会保存在内存中
4. 客户端会把从服务端获取到的配置在本地文件系统缓存一份
    * 在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置
5. 通过这种推拉结合的机制，以及内存和本地文件双缓存的方式，有效地保证了客户端的可用性

### 3.2.4 可用性场景举例

<table>
<thead>
<tr>
<th width="20%">场景</th>
<th width="20%">影响</th>
<th width="30%">降级</th>
<th width="30%">原因</th>
</tr>
</thead>
<tbody>
<tr>
<td>某台Config Service下线</td>
<td>无影响</td>
<td></td>
<td>Config Service无状态，客户端重连其它Config Service</td>
</tr>
<tr>
<td>所有Config Service下线</td>
<td>客户端无法读取最新配置，Portal无影响</td>
<td>客户端重启时，可以读取本地缓存配置文件。如果是新扩容的机器，可以从其它机器上获取已缓存的配置文件
</td>
<td></td>
</tr>
<tr>
<td>某台Admin Service下线</td>
<td>无影响</td>
<td></td>
<td>Admin Service无状态，Portal重连其它Admin Service</td>
</tr>
<tr>
<td>所有Admin Service下线</td>
<td>客户端无影响，Portal无法更新配置</td>
<td></td>
<td></td>
</tr>
<tr>
<td>某台Portal下线</td>
<td>无影响</td>
<td></td>
<td>Portal域名通过SLB绑定多台服务器，重试后指向可用的服务器</td>
</tr>
<tr>
<td>全部Portal下线</td>
<td>客户端无影响，Portal无法更新配置</td>
<td></td>
<td></td>
</tr>
<tr>
<td>某个数据中心下线</td>
<td>无影响</td>
<td></td>
<td>多数据中心部署，数据完全同步，Meta Server/Portal域名通过SLB自动切换到其它存活的数据中心</td>
</tr>
<tr>
<td>数据库全部宕机</td>
<td>客户端无影响，Portal无法更新配置</td>
<td>Config Service开启配置缓存后，对配置的读取不受数据库宕机影响</td>
<td></td>
</tr>
</tbody>
</table>

## 3.3 实时性

配置即『控制』，所以我们希望我们的控制指令能迅速、准确地传达到应用程序，我们来看看Apollo是如何实现实时性的。

![release-message-notification-design](/images/2018-07-29/release-message-notification-design.png)

上图简要描述了配置发布的大致过程：

1. 用户在Portal操作配置发布
2. Portal调用Admin Service的接口操作发布
3. Admin Service发布配置后，发送ReleaseMessage给各个Config Service
4. Config Service收到ReleaseMessage后，通知对应的客户端

### 3.3.1 发送ReleaseMessage的实现方式

Admin Service在配置发布后，需要通知所有的Config Service有配置发布，从而Config Service可以通知对应的客户端来拉取最新的配置。

从概念上来看，这是一个典型的消息使用场景，Admin Service作为producer发出消息，各个Config Service作为consumer消费消息。通过一个消息组件（Message Queue）就能很好的实现Admin Service和Config Service的解耦。

在实现上，考虑到Apollo的实际使用场景，以及为了尽可能减少外部依赖，我们没有采用外部的消息中间件，而是通过数据库实现了一个简单的消息队列。

实现方式如下：

1. Admin Service在配置发布后会往ReleaseMessage表插入一条消息记录，消息内容就是配置发布的AppId+Cluster+Namespace，参见[DatabaseMessageSender](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/DatabaseMessageSender.java)
2. Config Service有一个线程会每秒扫描一次ReleaseMessage表，看看是否有新的消息记录，参见[ReleaseMessageScanner](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/ReleaseMessageScanner.java)
3. Config Service如果发现有新的消息记录，那么就会通知到所有的消息监听器（[ReleaseMessageListener](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/ReleaseMessageListener.java)），如[NotificationControllerV2](https://github.com/ctripcorp/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/controller/NotificationControllerV2.java)，消息监听器的注册过程参见[ConfigServiceAutoConfiguration](https://github.com/ctripcorp/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/ConfigServiceAutoConfiguration.java)
4. NotificationControllerV2得到配置发布的AppId+Cluster+Namespace后，会通知对应的客户端

示意图如下：

<img src="/images/2018-07-29/release-message-design.png" alt="release-message-design" style="max-width: 400px;">

# 4. 如何让微服务更『智能』？

接下来我们来看一下结合配置中心，我们能做哪些有趣的事情，让微服务更智能。

## 4.1 开关

### 4.1.1 发布开关

发布开关一般用于发布过程中，比如：

1. 有些新功能依赖于其它系统的新接口，而其它系统的发布周期未必和自己的系统一致，可以加个发布开关，默认把该功能关闭，等依赖系统上线后再打开。
2. 有些新功能有较大风险，可以加个发布开关，上线后一旦有问题可以迅速关闭

需要注意的是，发布开关应该是短暂存在的（1-2周），一旦功能稳定后需要及时清除开关代码。

### 4.1.2 实验开关

实验开关通常用于对比测试或功能测试，比如：

1. A/B测试
    * 针对特定用户应用新的推荐算法
    * 针对特定百分比的用户使用新的下单流程
    * ![ab-test](/images/2018-07-29/ab-test.png) *[图片来源](https://www.search-factory.cn/ab-testing/)*
2. QA测试
    * 有些重大功能已经对外宣称在某年某日发布
    * 可以事先发到生产环境，只对内部用户打开，测试没问题后按时对全部用户开放
    * ![product-launch](/images/2018-07-29/product-launch.jpg) *[图片来源](http://www.businesses.com.au/marketing/418276-top-4-reasons-to-hire-promo-staff-for-your-product-launch-)*

实验开关应该也是短暂存在的，一旦实验结束了需要及时清除实验开关代码。

### 4.1.3 运维开关

运维开关通常用于提升系统稳定性，比如：

1. 大促前可以把一些非关键功能关闭来提升系统容量
2. 当系统出现问题时可以关闭非关键功能来保证核心功能正常工作

运维开关可能会长期存在，而且一般会涉及多个系统，所以需要提前规划。


## 4.2 服务治理

### 4.2.1 限流

服务就像高速公路一样，在正常情况下非常通畅，不过一旦流量突增（比如大促、遭受DDOS攻击）时，如果没有做好限流，就会导致系统整个被冲垮，所有用户都无法访问。

正常的高速公路

![normal-highway](/images/2018-07-29/normal-highway.jpg)

*[图片来源](https://uspirg.org/sites/pirg/files/cpn/USN-041817-A3-REPORT/highway-boondoggles-3.html)*

超出容量的高速公路

![traffic-jam-highway](/images/2018-07-29/traffic-jam-highway.jpg)

*[图片来源](https://abcnews.go.com/International/thousands-cars-stuck-beijing-traffic-jam-50-lane/story?id=34350370)*

所以我们需要限流机制来应对此类问题，一般的做法是在网关或RPC框架层添加限流逻辑，结合配置中心的动态推送能力实现动态调整限流规则配置。

### 4.2.2 黑白名单

对于一些关键服务，哪怕是在内网环境中一般也会对调用方有所限制，比如：

1. 有敏感信息的服务可以通过配置白名单来限制只有某些应用或IP才能调用
2. 某个调用方代码有问题导致超大量调用，对服务稳定性产生了影响，可以通过配置黑名单来暂时屏蔽这个调用方或IP

一般的做法是在RPC框架层添加校验逻辑，结合配置中心的动态推送能力来实现动态调整黑白名单配置。

## 4.3 数据库迁移

数据库的迁移也是挺普遍的，比如：原来使用的SQL Server，现在需要迁移到MySQL，这种情况就可以结合配置中心来实现平滑迁移：

1. 单写SQL Server，100%读SQL Server
2. 初始化MySQL
3. 双写SQL Server和MySQL，100%读SQL Server
4. 线下校验、补齐MySQL数据
5. 双写SQL Server和MySQL，90%读SQL Server，10%读MySQL
6. 双写SQL Server和MySQL，100%读MySQL
7. 单写MySQL，100%读MySQL
8. 切换完成

上述的读写开关和比例配置都可以通过配置中心实现动态调整。

![database-migration](/images/2018-07-29/database-migration.png)

## 4.4 动态日志级别

服务运行过程中，经常会遇到需要通过日志来排查定位问题的情况，然而这里却有个两难：

1. 如果日志级别很高（如：ERROR），可能对排查问题也不会有太大帮助
2. 如果日志级别很低（如：DEBUG），日常运行会带来非常大的日志量，造成系统性能下降

为了兼顾性能和排查问题，我们可以借助于日志组件和配置中心实现日志级别动态调整。

以Spring Boot和Apollo结合为例：

```java
  @ApolloConfigChangeListener
  private void onChange(ConfigChangeEvent changeEvent) {
    refreshLoggingLevels(changeEvent.changedKeys());
  }

  private void refreshLoggingLevels(Set<String> changedKeys) {
    boolean loggingLevelChanged = false;
    for (String changedKey : changedKeys) {
      if (changedKey.startsWith("logging.level.")) {
        loggingLevelChanged = true;
        break;
      }
    }

    if (loggingLevelChanged) {
      // refresh logging levels
      this.applicationContext.publishEvent(new EnvironmentChangeEvent(changedKeys));
    }
  }
```

详细样例代码可以参考：[https://github.com/ctripcorp/apollo-use-cases/tree/master/spring-cloud-logger](https://github.com/ctripcorp/apollo-use-cases/tree/master/spring-cloud-logger)

## 4.5 动态网关路由

网关的核心功能之一就是路由转发，而其中的路由信息也是经常会需要变化的，我们也可以结合配置中心实现动态更新路由信息。

以Spring Cloud Zuul和Apollo结合为例：

```java
  @ApolloConfigChangeListener
  public void onChange(ConfigChangeEvent changeEvent) {
    boolean zuulPropertiesChanged = false;
    for (String changedKey : changeEvent.changedKeys()) {
      if (changedKey.startsWith("zuul.")) {
        zuulPropertiesChanged = true;
        break;
      }
    }

    if (zuulPropertiesChanged) {
      refreshZuulProperties(changeEvent);
    }
  }

  private void refreshZuulProperties(ConfigChangeEvent changeEvent) {
    // rebind configuration beans, e.g. ZuulProperties
    this.applicationContext.publishEvent(new EnvironmentChangeEvent(changeEvent.changedKeys()));

    // refresh routes
    this.applicationContext.publishEvent(new RoutesRefreshedEvent(routeLocator));
  }
```

详细样例代码可以参考：[https://github.com/ctripcorp/apollo-use-cases/tree/master/spring-cloud-zuul](https://github.com/ctripcorp/apollo-use-cases/tree/master/spring-cloud-zuul)

## 4.6 动态数据源

数据库是应用运行过程中的一个非常重要的资源，承担了非常重要的角色。

在运行过程中，我们会遇到各种不同的场景需要让应用程序切换数据库连接，比如：数据库维护、数据库宕机主从切换等。

切换过程如下图所示：

![dynamic-datasource](/images/2018-07-29/dynamic-datasource.png)

以Spring Boot和Apollo结合为例：

```java
@Configuration
public class RefreshableDataSourceConfiguration {

  @Bean
  public DynamicDataSource dataSource(DataSourceManager dataSourceManager) {
    DataSource actualDataSource = dataSourceManager.createDataSource();
    return new DynamicDataSource(actualDataSource);
  }
}
```

```java
public class DynamicDataSource implements DataSource {
  private final AtomicReference<DataSource> dataSourceAtomicReference;

  public DynamicDataSource(DataSource dataSource) {
    dataSourceAtomicReference = new AtomicReference<>(dataSource);
  }

  // set the new data source and return the previous one
  public DataSource setDataSource(DataSource newDataSource){
    return dataSourceAtomicReference.getAndSet(newDataSource);
  }

  @Override
  public Connection getConnection() throws SQLException {
    return dataSourceAtomicReference.get().getConnection();
  }

  ...
}
```

```java
  @ApolloConfigChangeListener
  public void onChange(ConfigChangeEvent changeEvent) {
    boolean dataSourceConfigChanged = false;
    for (String changedKey : changeEvent.changedKeys()) {
      if (changedKey.startsWith("spring.datasource.")) {
        dataSourceConfigChanged = true;
        break;
      }
    }

    if (dataSourceConfigChanged) {
      refreshDataSource(changeEvent.changedKeys());
    }
  }

  private synchronized void refreshDataSource(Set<String> changedKeys) {
    try {
      // rebind configuration beans, e.g. DataSourceProperties
      this.applicationContext.publishEvent(new EnvironmentChangeEvent(changedKeys));

      DataSource newDataSource = dataSourceManager.createAndTestDataSource();
      DataSource oldDataSource = dynamicDataSource.setDataSource(newDataSource);
      asyncTerminate(oldDataSource);
    } catch (Throwable ex) {
      logger.error("Refreshing data source failed", ex);
    }
  }

```

详细样例代码可以参考：[https://github.com/ctripcorp/apollo-use-cases/tree/master/dynamic-datasource](https://github.com/ctripcorp/apollo-use-cases/tree/master/dynamic-datasource)

# 5. 最佳实践

## 5.1 公共组件的配置

公共组件是指那些发布给其它应用使用的客户端代码，比如RPC客户端、DAL客户端等。

这类组件一般是由单独的团队（如中间件团队）开发、维护，但是运行时是在业务实际应用内的，所以本质上可以认为是应用的一部分。

这类组件的特殊之处在于大部分的应用都会直接使用中间件团队提供的默认值，少部分的应用需要根据自己的实际情况对默认值进行调整。

比如数据库连接池的最小空闲连接数量（minimumIdle），出于对数据库资源的保护，DBA要求将全公司默认的minimumIdle设为1，对大部分的应用可能都适用，不过有些核心/高流量应用可能觉得太小，需要设为10。

针对这种情况，可以借助于Apollo提供的Namespace实现：

1.中间件团队创建一个名为`dal`的公共Namespace，设置全公司的数据库连接池默认配置

```properties
minimumIdle = 1
maximumPoolSize = 20
```

2.dal组件的代码会读取`dal`公共Namespace的配置

3.对大部分的应用由于默认配置已经适用，所以不用做任何事情

4.对于少量核心/高流量应用如果需要调整minimumIdle的值，只需要关联`dal`公共Namespace，然后对需要覆盖的配置做调整即可，调整后的配置仅对该应用自己生效

```properties
minimumIdle = 10
```

<img src="/images/2018-07-29/public-namespace.png" alt="public-namespace" style="max-width: 720px;">

通过这种方式的好处是不管是中间件团队，还是应用开发，都可以灵活地动态调整公共组件的配置。

## 5.2 灰度发布

对于重要的配置一定要做灰度发布，先在一台或多台机器上生效后观察效果，如果没有问题再推给所有的机器。

对于公共组件的配置，建议先在一个或多个应用上生效后观察效果，没有问题再推给所有的应用。

![canary-release](/images/2018-07-29/canary-release.png)

*[图片来源](http://www.appadhoc.com/blog/one-page-canary-release-test/)*

## 5.3 发布审核

生产环境建议启用发布审核功能，简单而言就是如果某个人修改了配置，那么必须由另一个人审核后才可以发布，以避免由于头脑不清醒、手一抖之类的造成生产事故。

![approval](/images/2018-07-29/approval.png)

*[图片来源](https://livinator.com/5-tips-to-get-easier-hoa-approval-for-your-home-renovations/)*

# 6. 结语

本文主要介绍了以下几方面：

1. 为什么需要配置中心？
	* 配置即『控制』
	* 配置需要治理
	* 微服务带来的配置复杂性
2. 配置中心的一般模样
	* 以Apollo为例子，介绍了配置中心所具备的特征
	* 介绍了Apollo是如何实现高可用和实时性的
3. 如何让微服务更『智能』？
	* 通过几个案例，分享了如何借助于配置中心使微服务更『智能』
4. 配置中心的最佳实践
   * 公共组件的配置
   * 灰度发布
   * 发布审核

最后，希望大家在平时工作中都能用好配置中心，更好地服务于业务场景，使微服务更『智能』，实现从青铜到王者的跨越！