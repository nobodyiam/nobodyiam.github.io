---
layout:     post
title:      Spring DeferredResult占用数据库连接问题解决
date:       2016-05-06 14:00:00 +0800
summary:    最近在项目中使用了Spring DeferredResult来实现Http长连接功能，不过在压力测试的时候发现数据库连接很快被占满，从而导致无法响应更多的请求。本文讲述了这个问题产生的原因以及解决办法。
categories:
---
最近在项目中尝试采用了<a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html" target="_blank">Spring DeferredResult</a>来实现Http长连接的功能，不过在压力测试的时候发现数据库连接很快被占满，从而导致无法响应更多的请求。本文讲述了这个问题产生的原因以及解决办法。

## 1. 使用场景
最近的<a href="https://github.com/ctripcorp/apollo" target="_blank">配置中心项目</a>有一个需求是当用户有操作配置更新时，客户端能实时感知并应用最新的值。所以我们需要在客户端和服务端之间建立一个长连接，当有用户操作配置更新时，通过这个长连接及时通知到客户端。

出于简单起见，我们决定采用基于Http的长连接，并通过Spring的DeferredResult来实现。
大致思路是对每一个客户端的长连接，生成一个DeferredResult对象，并且关联客户端所关心的配置组（这一步会访问数据库）。当对应的配置组有变化的时候会发出消息，通过消息找到对应的DeferredResult对象，并通过调用setResult来通知到客户端。

示例代码如下：

<script src="https://gist.github.com/nobodyiam/cfe3f3b1db782204f25b6408151cf841.js"></script>

## 2. 遇到的问题
通过上面的代码示例，可以看到，其实逻辑很简单，代码也很轻量，所以想当然觉得跑个压力测试是毫无问题的。可事实是当同时发出200个请求的时候，有100个请求竟然直接失败了。通过查看日志，发现如下错误日志：

>[ERROR] org.hibernate.engine.jdbc.spi.SqlExceptionHelper - [http-nio-8080-exec-51] Timeout: Pool empty. Unable to fetch a connection in 30 seconds, 
none available[size:100; busy:100; idle:0; lastwait:30000].

初遇到这个问题，还是挺纳闷的，不过在反复尝试几次之后，都是同样的结果，就感觉里面肯定有些可以挖掘的问题。

## 3. 问题分析
初步分析一下错误日志，数据库连接池共100个连接，提示的错误是当前100个都处于busy状态，没有idle状态的数据库连接。

所以导致100个请求失败的原因是另外100个请求一直占着数据库连接没有释放？

在经过多番求证和源码分析后，发现确实如此，而且这是Spring的默认行为。

Spring JPA提供了<a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/orm/jpa/support/OpenEntityManagerInViewInterceptor.html" target="_blank"> OpenEntityManagerInViewInterceptor</a>，它会在每个请求处理线程开始的时候绑定一个EntityManager，从而使整个线程处理过程中都可以访问到数据库，最后当线程处理完成之后，再把绑定的EntityManager关闭掉，在EntityManager关闭的过程中会同时把数据库连接放回数据库连接池中。所以在一般情况下都是没有问题的。

OpenEntityManagerInViewInterceptor.afterCompletion示例代码：

<script src="https://gist.github.com/nobodyiam/3db4bcaa5a694d26e2fcd08678cf39e2.js"></script>

不过，由于我使用了DeferredResult，情况就变得有一些不一样了。通过查看<a href="https://github.com/spring-projects/spring-framework/blob/v4.2.6.RELEASE/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java#L963" target="_blank">DispatcherServlet</a>的源码，我们发现Spring对于异步请求的处理有一些不一样。

DispatcherServlet.doDispatch示例代码：

<script src="https://gist.github.com/nobodyiam/4701e83e0a118167098eba63644b15e0.js"></script>

OpenEntityManagerInViewInterceptor.afterCompletion方法会通过上图示例代码中的processDispatchResult调用到。

但需要注意的是，如果当前请求是一个异步请求，而且请求还没最终结束的话（asyncManager.isConcurrentHandlingStarted()返回true），doDispatch方法在执行processDispatchResult之前就return掉了！

所以原因已经很清楚了，对于客户端的长连接，在请求线程开始时，会通过OpenEntityManagerInViewInterceptor绑定一个EntityManager，在程序执行过程中由于使用到了数据库，所以EntityManager就会占用一个数据库连接。

由于Spring DispatcherServlet的默认逻辑，这个数据库连接只有在异步请求真正返回给客户端的时候才会释放回连接池。考虑到我这里的长连接测试场景，就意味着只有前100个请求能被处理，后100个在获取数据库连接的时候就失败了。

## 4. 解决办法
其实仔细想想，Spring采取这种默认行为还是有一定意义的，因为它需要保证在异步请求的整个生命周期中都持有EntityManager，所以它采取的策略是在异步请求最终处理完成的时候再做关闭EntityManager操作。

不过对于我这个长连接场景，就不太适合了。有两个主要原因：

1. 我的长连接时间很长，对于大部分请求可能都要数小时以上才会返回。在这么长的一段时间内一直占用着数据库连接是不合理的。
2. 对于我的长连接请求，其实只在请求一开始才需要用到数据库连接，之后就不再需要使用数据库了

所以，我最后采用了一个比较简单的workaround：在长连接请求使用完数据库之后主动关闭EntityManager。代码示例如下：

<script src="https://gist.github.com/nobodyiam/e9d995593087f7bd59a96a01c32d198a.js"></script>

<script src="https://gist.github.com/nobodyiam/da0eecbfe076f8e87349f836a8361521.js"></script>

通过这个简单的workaround，就解决了DeferredResult请求长时间占用数据库连接的问题。不过其实这个感觉还是有一点hack的味道，希望后续能找到更好的解决方法。