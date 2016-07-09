---
layout:     post
title:      Apollo配置中心介绍
date:       2016-07-09 14:00:00 +0800
summary:    Apollo是携程框架研发团队开发的开源配置中心系统，本文主要对Apollo的核心概念和使用方法做了着重介绍。
categories:
---

# 1、What is Apollo

## 1.1 Apollo简介

Apollo是携程框架研发团队开发的配置解决方案，提供了配置中心管理界面和客户端（.Net和Java）。

Apollo支持4个维度管理Key-Value格式的配置：
	
1. application (应用)
2. environment (环境)
3. cluster (集群)
4. namespace (命名空间)

同时，Apollo基于开源模式开发，开源地址：<a href="https://github.com/ctripcorp/apollo" target="_blank">https://github.com/ctripcorp/apollo</a>

## 1.2 配置基本概念

既然是Apollo定位于配置中心，那么在这里有必要先简单介绍一下什么是配置。

按照我们的理解，配置有以下几个属性：

* **配置是独立于程序的只读变量**
	* 配置首先是独立于程序的，同一份程序在不同的配置下会有不同的行为。
	* 其次，配置对于程序是只读的，程序通过读取配置来改变自己的行为，但是程序不应该去改变配置。
	* 常见的配置有：DB Connection Str、Thread Pool Size、Buffer Size、Request Timeout、Feature Switch、Server Urls等。

* **配置伴随应用的整个生命周期**
	* 配置贯穿于应用的整个生命周期，应用在启动时通过读取配置来初始化，在运行时根据配置调整行为。

* **配置可以有多种加载方式**
	* 配置也有很多种加载方式，常见的有程序内部hard code，配置文件，环境变量，启动参数，基于数据库等

* **配置需要治理**
	* 权限控制
		* 由于配置能改变程序的行为，不正确的配置甚至能引起灾难，所以对配置的修改必须有比较完善的权限控制
	* 不同环境、集群配置管理
		* 同一份程序在不同的环境（开发，测试，生产）、不同的集群（如不同的数据中心）经常需要有不同的配置，所以需要有完善的环境、集群配置管理
	* 框架类组件配置管理
		* 还有一类比较特殊的配置 - 框架类组件配置，比如CAT客户端的配置。
		* 虽然这类框架类组件是由其他团队开发、维护，但是运行时是在业务实际应用内的，所以本质上可以认为框架类组件也是应用的一部分。
		* 这类组件对应的配置也需要有比较完善的管理方式。

# 2、Why Apollo

正是基于配置的特殊性，所以Apollo从设计开始就立志于成为一个有治理能力的配置发布平台，并且提供了以下的特性：

* **集中化配置管理**
	* Apollo提供了一个统一界面集中式管理不同环境（environment）、不同集群（cluster）、不同命名空间（namespace）的配置。

* **版本发布管理**
	* 所有的配置发布都有版本概念，从而可以方便的支持配置的回滚。

* **客户端实时更新**
	* 用户在Apollo修改完配置并发布后，客户端能实时（1秒）接收到最新的配置，并通知到应用程序。

* **数据中心、业务集群切换**
	* 通过配置不同的cluster，可以非常方便的支持同一个应用在不同集群的实例使用不同的配置。

* **授权、审核、审计**
	* 应用和配置的管理都有完善的权限管理机制，对配置的管理还分为了编辑和发布两个环节，从而减少人为的错误。
	* 所有的操作都有审计日志，可以方便的追踪问题。

* **支持.Net，Java应用**
	* 考虑到携程目前主要还是Java和.Net应用为主，所以Apollo提供了这两个语言的客户端。（.Net客户端暂未开源）
	* 不过客户端和服务端的通讯协议基于Http，所以可以很方便的开发出针对其它语言的客户端。

# 3、Apollo at a glance

## 3.1 基础模型

如下即是Apollo的基础模型：

1. 用户在配置中心对配置进行修改并发布
2. 配置中心通知Apollo客户端有配置更新
3. Apollo客户端从配置中心拉取最新的配置、更新本地配置并通知到应用

![basic-architecture](/images/2016-07-09/basic-architecture.png)

## 3.2 添加/修改配置项

用户可以通过配置中心界面方便的添加/修改配置项

![edit-item](/images/2016-07-09/edit-item.png)

## 3.3 发布配置

通过配置中心发布配置

![publish-items](/images/2016-07-09/publish-items.png)

## 3.4 客户端获取配置

配置发布后，就能在客户端获取到了，以Java为例，获取配置的示例代码如下。更多客户端使用说明请参见<a href="https://github.com/ctripcorp/apollo/blob/master/apollo-client/README.md" target="_blank">Java客户端使用文档</a>。

<script src="https://gist.github.com/nobodyiam/3d6753b3839bee3c32bd0373c037b2cc.js"></script>

## 3.5 客户端监听配置变化

通过上述获取配置代码，应用就能实时获取到最新的配置了。

不过在某些场景下，应用还需要在配置变化时获得通知，比如数据库连接的切换等，所以Apollo还提供了监听配置变化的功能，Java示例如下：

<script src="https://gist.github.com/nobodyiam/723aa16ea33f552f794806c5f62648b5.js"></script>

# 4、Apollo in depth

通过上面的介绍，相信大家已经对Apollo有了一个初步的了解，并且相信已经覆盖到了大部分的使用场景。

接下来会主要介绍Apollo的cluster管理（集群）、namespace管理（命名空间）和对应的配置获取规则。

## 4.1 Core Concepts

在介绍高级特性前，我们有必要先来了解一下Apollo中的几个核心概念：

1. **application (应用)**
	* 使用配置的应用，每个应用管理自己应用所用到的所有配置。
	* 应用需要有唯一标识 - appId，appId通过app.properties指定，具体信息请参见<a href="https://github.com/ctripcorp/apollo/blob/master/apollo-client/README.md" target="_blank">Java客户端使用文档</a>。

2. **environment (环境)**
	* 配置对应的环境，同一个配置在不同的环境可以有不一样的值。
	* 应用在运行时需要指定环境，环境可以通过System Property或server.properties指定，具体信息请参见<a href="https://github.com/ctripcorp/apollo/blob/master/apollo-client/README.md" target="_blank">Java客户端使用文档</a>。

3. **cluster (集群)**
	* 一个应用下不同实例的分组，比如典型的可以按照数据中心分，把A机房的应用实例分为一个集群，把B机房的应用实例分为另一个集群。
	* 对不同的cluster，同一个配置可以有不一样的值，如数据库地址。
	* 应用在运行时可以指定集群，集群可以通过System Property或server.properties指定，具体信息请参见<a href="https://github.com/ctripcorp/apollo/blob/master/apollo-client/README.md" target="_blank">Java客户端使用文档</a>。

4. **namespace (命名空间)**
	* 一个应用下不同配置的分组，每个应用在Apollo创建后都会有一个默认的namespace - application。
	* 应用也可以引入公共组件的配置namespace，如DAL，Hermes等，一旦引入过来，就可以对公共组件的配置做调整，如hermes producer的batch size。

## 4.2 自定义Cluster

比如我们有应用在A数据中心和B数据中心都有部署，那么如果希望两个数据中心的配置不一样的话，我们可以通过新建cluster来解决。

### 4.2.1 新建Cluster

新建Cluster只有项目的管理员才有权限，管理员可以在页面左侧看到“添加集群”按钮。

![create-cluster](/images/2016-07-09/create-cluster.png)

点击后就进入到集群添加页面，一般情况下可以按照数据中心来划分集群，如SHAJQ、SHAOY等。

不过也支持自定义集群，比如可以为A机房的某一台机器和B机房的某一台机创建一个集群，使用一套配置。

![create-cluster-detail](/images/2016-07-09/create-cluster-detail.png)

### 4.2.2 在Cluster中添加配置并发布

集群添加成功后，就可以为该集群添加配置了，首选需要按照下图所示切换到SHAJQ集群，之后配置添加流程和[3.2添加/修改配置项](#section-2)一样，这里就不再赘述了。

![cluster-created](/images/2016-07-09/cluster-created.png)

### 4.2.3 指定应用实例所属的Cluster

Apollo会默认使用应用实例所在的数据中心作为cluster，所以如果两者一致的话，不需要额外配置。

如果cluster和数据中心不一致的话，那么就需要通过System Property方式来指定运行时cluster：

* -Dapollo.cluster=SomeCluster
* 这里注意`apollo.cluster`为全小写

## 4.3 自定义Namespace

如果应用有公共组件（如hermes-producer，cat-client等）供其它应用使用，就需要通过自定义namespace来实现公共组件的配置。

### 4.3.1 新建Namespace

以hermes-producer为例，需要先新建一个namespace，新建namespace只有项目的管理员才有权限，管理员可以在页面左侧看到“添加Namespace”按钮。

![create-namespace](/images/2016-07-09/create-namespace.png)

点击后就进入namespace添加页面，Apollo会把应用所属的部门作为namespace的前缀，如FX。

![create-namespace-detail](/images/2016-07-09/create-namespace-detail.png)

### 4.3.2 关联到环境和集群

Namespace创建完，需要选择在哪些环境和集群下使用

![link-namespace-detail](/images/2016-07-09/link-namespace-detail.png)

### 4.3.3 在Namespace中添加配置项

接下来在这个新建的namespace下添加配置项

![add-item-in-new-namespace](/images/2016-07-09/add-item-in-new-namespace.png)

添加完成后就能在FX.Hermes.Producer的namespace中看到配置。

![item-created-in-new-namespace](/images/2016-07-09/item-created-in-new-namespace.png)

### 4.3.4 发布namespace的配置

![publish-items-in-new-namespace](/images/2016-07-09/publish-items-in-new-namespace.png)

### 4.3.5 客户端获取Namespace配置

对自定义namespace的配置获取，稍有不同，需要程序传入namespace的名字。更多客户端使用说明请参见<a href="https://github.com/ctripcorp/apollo/blob/master/apollo-client/README.md" target="_blank">Java客户端使用文档</a>。

<script src="https://gist.github.com/nobodyiam/32e77d92849c13d035943e249c82ddcf.js"></script>

### 4.3.6 客户端监听Namespace配置变化

<script src="https://gist.github.com/nobodyiam/06bd94f5d253525ca9d3b37189229271.js"></script>

## 4.4 配置获取规则

在有了cluster概念后，配置的规则就显得重要了。

比如应用部署在A机房，但是并没有在Apollo新建cluster，这个时候Apollo的行为是怎样的？

或者在运行时指定了cluster=SomeCluster，但是并没有在Apollo新建cluster，这个时候Apollo的行为是怎样的？

接下来就来介绍一下配置获取的规则。

### 4.4.1 应用自身配置的获取规则

当应用使用下面的语句获取配置时，我们称之为获取应用自身的配置，也就是应用自身的application namespace的配置。

	Config config = ConfigService.getAppConfig();

对这种情况的配置获取规则，简而言之如下：

1. 首先查找运行时cluster的配置（通过apollo.cluster指定）
2. 如果没有找到，则查找数据中心cluster的配置
3. 如果还是没有找到，则返回默认cluster的配置

图示如下：

![application-config-precedence](/images/2016-07-09/application-config-precedence.png)

所以如果应用部署在A数据中心，但是用户没有在Apollo创建cluster，那么获取的配置就是默认cluster（default）的。

如果应用部署在A数据中心，同时在运行时指定了SomeCluster，但是没有在Apollo创建cluster，那么获取的配置就是A数据中心cluster的配置，如果A数据中心cluster没有配置的话，那么获取的配置就是默认cluster（default）的。

### 4.4.2 公共组件配置的获取规则

以***FX.Hermes.Producer***为例，hermes producer是hermes发布的公共组件。当使用下面的语句获取配置时，我们称之为获取公共组件的配置。

	Config config = ConfigService.getConfig("FX.Hermes.Producer");

对这种情况的配置获取规则，简而言之如下：

1. 首先获取当前应用下的***FX.Hermes.Producer*** namespace的配置
2. 然后获取hermes应用下***FX.Hermes.Producer*** namespace的配置
3. 上面两部分配置的并集就是最终使用的配置，如有key一样的部分，以当前应用优先

图示如下：

![public-namespace-config-precedence](/images/2016-07-09/public-namespace-config-precedence.png)

通过这种方式，就实现了对框架类组件的配置管理，框架组件提供方提供配置的默认值，应用如果有特殊需求，可以自行覆盖。

## 4.5 客户端实现原理

![client-architecture](/images/2016-07-09/client-architecture.png)

上图简要描述了Apollo客户端的实现原理：

1. 客户端会定时从Apollo配置中心服务端拉取应用的最新配置，并保存在内存中
2. 客户端还和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。（通过Http Long Polling实现）
3. 客户端会把从服务端获取到的配置在本地文件系统缓存一份
	* 在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置
4. 应用程序可以从Apollo客户端获取最新的配置、订阅配置更新通知

# 5、Future

Apollo还在持续的开发迭代中，未来我们会提供更多的功能，大致方向如下：

* **版本回滚**
	* 提供版本回滚的功能，从而当用户发现配置有问题后，可以方便地回退到上一个版本。

* **灰度发布**
	* 支持配置的灰度发布，比如点了发布后，只对部分应用实例生效，等观察一段时间没问题后再推给所有应用实例。

* **配置运行时监控**
	* 提供更多运行时监控的工具，比如能看到哪些应用实例在使用哪些cluster和namespace的配置。

* **应用配置迁移工具**
	* 提供迁移工具，帮助已有的应用可以快速的把配置迁移到Apollo中。

* **作为平台接入其它配置工具**
	* 把配置作为服务提供出来，从而支持配置的多样化使用场景。
	* 比如DAL的使用方可以在DAL平台做数据库相关信息的配置，在DAL做完相关的校验和验证后，输入到Apollo，最后通知到应用。

# 6、Contribute to Apollo

Apollo是作为一个开源项目开发的，目前开发/测试/产品/运维总共两个人，所以也非常希望能有更多的力量投入进来。

目前服务端开发使用的是Java，基于Spring Cloud和Spring Boot框架。客户端目前提供了Java和.Net两种实现。

Github地址：<a href="https://github.com/ctripcorp/apollo" target="_blank">https://github.com/ctripcorp/apollo</a>

欢迎大家发起Pull Request！
