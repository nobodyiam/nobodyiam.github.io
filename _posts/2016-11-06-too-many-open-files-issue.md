---
layout:     post
title:      Tomcat应用报Too many open files的故障排查
date:       2016-11-06 15:00:00 +0800
summary:    最近业务有一个应用突然报大量错误(Too many open files)导致系统出各种问题，由于在时间上是用户正好在Apollo配置中心操作了配置发布，所以第一时间就找到我们一起帮忙定位问题。本文详细介绍了问题的排查过程，问题产生的原因以及后续优化的一些措施。
categories:
---
最近业务有一个应用突然报大量错误(Too many open files)导致系统出问题，由于在时间上是用户正好在Apollo配置中心操作了配置发布，所以第一时间就找到我们一起帮忙定位问题。本文详细介绍了问题的排查过程，问题产生的原因以及后续优化的一些措施。

# 1. 故障现象
开发在生产环境通过Apollo修改了配置并操作了发布后，发现服务开始大量报错。

通过查看日志，发现redis无法获取到连接了，由于业务依赖于redis提供缓存服务，所以导致接口大量报错。


	Caused by: redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool
	         at redis.clients.util.Pool.getResource(Pool.java:50)
	         at redis.clients.jedis.JedisPool.getResource(JedisPool.java:99)
	         at credis.java.client.entity.RedisServer.getJedisClient(RedisServer.java:219)
	         at credis.java.client.util.ServerHelper.execute(ServerHelper.java:34)
	         ... 40 more
	Caused by: redis.clients.jedis.exceptions.JedisConnectionException: java.net.SocketException: Too many open files
	         at redis.clients.jedis.Connection.connect(Connection.java:164)
	         at redis.clients.jedis.BinaryClient.connect(BinaryClient.java:82)
	         at redis.clients.jedis.BinaryJedis.connect(BinaryJedis.java:1641)
	         at redis.clients.jedis.JedisFactory.makeObject(JedisFactory.java:85)
	         at org.apache.commons.pool2.impl.GenericObjectPool.create(GenericObjectPool.java:868)
	         at org.apache.commons.pool2.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:435)
	         at org.apache.commons.pool2.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:363)
	         at redis.clients.util.Pool.getResource(Pool.java:48)
	         ... 43 more
	Caused by: java.net.SocketException: Too many open files
	         at java.net.Socket.createImpl(Socket.java:447)
	         at java.net.Socket.getImpl(Socket.java:510)
	         at java.net.Socket.setReuseAddress(Socket.java:1396)
	         at redis.clients.jedis.Connection.connect(Connection.java:148)
	         ... 50 more

由于该服务是基础服务，有很多上层应用依赖它，所以也导致了一系列连锁反应。情急之下，把所有应用的tomcat都重启了一遍，故障恢复。

# 2. 初步分析
我们得到通知的时候，故障已经恢复，也没了现场。所以只能依赖于日志和[CAT](https://github.com/dianping/cat)监控来尝试找到一些线索。

从CAT监控上看，该应用集群共20台机器，不过在Apollo客户端获取到最新配置，准备通知应用该次配置的变化详情时，只有5台通知成功，15台通知失败。

其中15台通知失败机器的JVM似乎有些问题，无法加载类（NoClassDefFoundError），错误信息被Apollo的代码catch住并记录到了CAT。

**5台成功的信息如下：**

![config-change-5-successful](/images/2016-11-06/config-change-5-successful.png)

**15台失败的如下：**

![config-change-15-failed](/images/2016-11-06/config-change-15-failed.png)

	Caused by: java.lang.ClassNotFoundException: com.ctrip.framework.apollo.model.ConfigChange
	at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1718)
	at org.apache.catalina.loader.WebappClassLoader.loadClass(WebappClassLoader.java:1569)
	... 16 more

> Apollo在配置更新后，会计算配置的变化并通知到应用。配置的变化会通过`ConfigChange`对象存储。
> 
> 由于该应用是启动后第一次配置变化，所以`ConfigChange`类是第一次使用到，由于JVM的懒加载机制，这时会触发一次类加载过程。

这里就有一个疑问来了，为啥JVM会无法加载类？这个类`com.ctrip.framework.apollo.model.ConfigChange`是打在apollo-client的jar包中的，和Apollo其它的类是在同一个jar包中，不应该出现NoClassDefFoundError的情况。

而且，碰巧的是，后续redis连接出错的也正是这15台报了NoClassDefFoundError的机器。

联想到前面的报错`Too many open files`，会不会也是由于文件句柄数不够，所以导致JVM无法从文件系统读取jar包，从而导致`NoClassDefFoundError `？

# 3. 故障原因
关于该应用出现的问题，种种迹象表明那个时段应该是进程句柄数不够引起的，例如无法从本地加载文件，无法建立redis连接，无法发起网络请求等等。

由于前一阵框架的一个应用也出现了这个问题，当时发现老机器的`Max Open Files`设置是65536，但是新的机器上的`Max Open Files`都被误设置为4096了。

虽然后来运维统一修复了这个设置问题，但是需要重启才会生效。所以目前生产环境应该还有相当大一部分机器的`Max Open Files`是4096的。

所以，我们登陆到其中一台出问题的机器上去查看是否存在这个问题。但是出问题的应用已经重启，无法获取到应用当时的情况。不过好在该机器上还部署了另外的应用，pid为16112。通过查看/proc/16112/limits文件，看到里面的`Max Open Files`是4096。

![max-open-files-4096](/images/2016-11-06/max-open-files-4096.png)

所以有理由相信应用出问题的时候，它的`Max Open Files`也是4096，一旦当时的句柄数达到4096的话，就会导致后续所有的IO都出现问题。

所以故障原因算是找到了，是由于`Max Open Files`的设置太小，一旦进程打开的文件句柄数达到4096，后续的所有请求（文件IO，网络IO）都会失败。

由于该应用依赖了redis，所以一旦一段时间内无法连接redis，就会导致请求大量超时，造成请求堆积，进入恶性循环。（好在SOA框架有熔断和限流机制，所以问题影响只持续了几分钟）

# 4. 疑团重重
故障原因算是找到了，各方似乎对这个答案还算满意。不过还是有一个疑问一直在心头萦绕，为啥故障发生时间这么凑巧，就发生在用户通过Apollo发布配置后？

为啥在Apollo配置发布前，系统打开的文件句柄还小于4096，在Apollo配置发布后就超过了？

难道Apollo配置发布后会大量打开文件句柄？

## 4.1 Apollo客户端代码分析
通过对Apollo客户端代码分析，Apollo在推送配置后，客户端做了以下几件事情：

1. 之前断开的http long polling会重新连接
2. 会有一个异步task去服务器获取最新配置
3. 获取到最新配置后会写入本地文件
4. 写入本地文件成功后，会计算diff并通知到应用

从上面的步骤可以看出，第1步会重新建立之前断开的连接，所以不算新增，第2步和第3步会短暂的各新增一个文件句柄（一次网络请求和一次本地IO），不过执行完后都会释放掉。

## 4.2 Web应用尝试重现
代码看了几遍也没看出问题，于是尝试重现问题，所以我临时找了另一个应用(hermes portal)的测试环境做了一个测试，选hermes portal做测试的原因是它和出问题的应用非常相似：

1. 都是Web应用
2. 都运行了有一段时间（超过一周）
3. 在上一次启动后，配置都没有发布过（也就是说ConfigChange这个类还没被加载，所以能重现加载类的过程）

### 4.2.1 测试结果

1. Hermes portal的pid是3823
2. 在执行配置发布前的进程总打开文件数是511，其中打开的jar文件数是310
3. 在执行配置发布后的一秒内，总打开文件数是515，其中打开的jar文件数是314（由于时间短暂，没有来得及捕获新增的jar文件是哪些）
4. 稍等2-3秒后，总打开文件数是511，其中打开的jar文件数是310（恢复之前的数量了）

**Apollo配置发布前：**

![hermes-portal-pid](/images/2016-11-06/hermes-portal-pid.png)

![hermes-portal-lsof](/images/2016-11-06/hermes-portal-lsof.png)

![hermes-portal-lsof-jar](/images/2016-11-06/hermes-portal-lsof-jar.png)

**Apollo配置发布后：**

![hermes-portal-lsof-after-apollo](/images/2016-11-06/hermes-portal-lsof-after-apollo.png)

![hermes-portal-lsof-jar-after-apollo](/images/2016-11-06/hermes-portal-lsof-jar-after-apollo.png)

**过了2-3秒后：**

![hermes-portal-lsof-after-apollo](/images/2016-11-06/hermes-portal-lsof-after-apollo-several-seconds.png)

![hermes-portal-lsof-jar-after-apollo](/images/2016-11-06/hermes-portal-lsof-jar-after-apollo-several-seconds.png)

> 这里的重现实验其实有一个问题，虽然hermes-portal也是通过tomcat启动的应用，但它使用的是tomcat-embed，不是独立的tomcat。

## 4.3 Console程序尝试重现：

在本机也尝试重现并更细粒度的捕获进程打开文件信息，所以在本地起了一个apollo demo应用（console application）读取apollo配置，然后通过bash脚本实时打印文件信息。
 
{% highlight sh%}
for i in {1..1000}
do
  lsof -p 91742 > /tmp/20161101/$i.log
  sleep 0.01
done
{% endhighlight %}

不过在本地并没有发现有新加载jar包的情况（或许是因为启动时间比较短，还有缓存？）。

本地能观察到的区别是在Apollo配置发布后，会在很短的时间内多两个TCP连接，其中一个保持了0.02秒，另一个保持了2秒左右也消失了。
 
	java    91742 Jason   67u    IPv6 0x82565eeb0b384f93       0t0      TCP [::127.0.0.1]:59839->[::127.0.0.1]:lnvpoller (CLOSED)
	java    91742 Jason   63u    IPv6 0x82565eeb150f2a13       0t0      TCP localhost:59787->localhost:commplex-link (ESTABLISHED)

从以上现象看来，Apollo在配置发布后并不会大量增加文件句柄数，虽然洗清了Apollo客户端的嫌疑，但是问题的答案到底是什么呢？该如何解释观测到的种种现象呢？

# 5. 柳暗花明
尝试自己重现问题无果后，只剩下最后一招了 - 通过应用的程序直接重现问题。

为了不影响应用，我把应用程序的war包连带使用的tomcat在测试环境又独立部署了一份。不想竟然很快就发现了导致问题的原因。

原因是tomcat对webapp有一套自己的WebappClassLoader，它在启动的过程中会打开jar文件，但是过一段时间就会全部关闭来释放资源。

然而如果后面需要加载某个新的class的时候，会把之前所有的jar文件全部打开一遍，然后再从中找到对应的jar来加载。加载完后过一段时间会再一次全部释放掉。**所以应用依赖的jar包越多，打开的文件句柄也会越多。**

同时，在tomcat的源码中也找到了上述[WebappClassLoader](http://atetric.com/atetric/javadoc/org.apache.tomcat/tomcat-catalina/7.0.72/src-html/org/apache/catalina/loader/WebappClassLoaderBase.html)的逻辑。

>前面的重现实验最大的问题就是没有完全复现应用出问题时的场景，如果当时就直接测试了独立安装的tomcat（而不是embedded tomcat），问题原因就能更早的发现。

## 5.1 重现环境分析

### 5.1.1 Tomcat刚启动完

刚启动完，进程打开的句柄数是443。
{% highlight sh%}
lsof -p 31188 | wc -l
443
{% endhighlight %}
 
### 5.1.2 Tomcat启动完过了10分钟左右
启动完过了约10分钟，再次查看，发现只剩192个了。仔细比较了一下其中的差异，原来是由于WEB-INF/lib下的jar包句柄全释放了。

{% highlight sh%}
lsof -p 31188 | wc -l
192

lsof -p 31188 | grep "WEB-INF/lib" | wc -l
0
{% endhighlight %}
 
### 5.1.3 Apollo配置发布后2秒左右
然后通过Apollo做了一次配置发布，再次查看，发现一下子涨到422了。其中的差异是之前释放的WEB-INF/lib下的jar包句柄又出现了，有228个之多。
{% highlight sh%}
lsof -p 31188 | wc -l
422
 
lsof -p 31188 | grep "WEB-INF/lib" | wc -l
228
{% endhighlight %}
 
### 5.1.4 Apollo配置发布30秒后
过了约30秒后，WEB-INF/lib下的jar包句柄又全部释放了。
{% highlight sh%}
lsof -p 31188 | wc -l
194
 
lsof -p 31188 | grep "WEB-INF/lib" | wc -l
0
{% endhighlight %}

## 5.2 Tomcat WebappClassLoader逻辑
通过查看Tomcat的[WebappClassLoader](http://atetric.com/atetric/javadoc/org.apache.tomcat/tomcat-catalina/7.0.72/src-html/org/apache/catalina/loader/WebappClassLoaderBase.html)逻辑，也印证了我们的实验结果。

### 5.2.1 加载类逻辑
首先会打开所有的jar文件，然后遍历找到对应的jar去加载

![tomcat-find-resource-internal](/images/2016-11-06/tomcat-find-resource-internal.png)

![tomcat-open-jars](/images/2016-11-06/tomcat-open-jars.png)

### 5.2.2 关闭jar文件逻辑
还有一个后台线程会定期去做文件的关闭动作

![tomcat-background-process](/images/2016-11-06/tomcat-background-process.png)

![tomcat-close-jars](/images/2016-11-06/tomcat-close-jars.png)

## 5.3 故障场景分析
对于Apollo配置推送的场景，会使用到一个之前没有用到的class: `com.ctrip.framework.apollo.model.ConfigChange`，所以会触发tomcat类加载，并进而打开所有的jar文件，从而会导致在很短的时间内进程句柄数升高。（对该应用而言，之前测试下来的数字是228）。

虽然现在无从知晓该应用在出问题前总的文件句柄数，但是从CAT监控可以看到光TCP连接数就在3200+了，再加上一些jdk加载的类库（这部分tomcat不会释放）和本地文件，应该离4096的上限相差不多了。所以这时候如果Tomcat再一下子打开本地228个文件，自然就很容易导致`Too many open files`的问题了。

![app-tcp-connections](/images/2016-11-06/app-tcp-connections.png)
 
回到应用的场景，观测下来在Apollo推送过后2秒开始，jedis频繁报由于`Too many open files`无法连接Redis，从而导致请求超时增多，并进而导致应用整体拖慢，陷入恶性循环。

>这里尚不知晓为啥jedis需要频繁的连接Redis，如果是长连接应该就不会出现这个问题了吧？

# 6. 总结

## 6.1 问题产生原因
所以，分析到这里，整件事情的脉络就清晰了：

1. 应用的Max Open Files设置成了4096
2. 应用自身的文件句柄数较高，已经接近了4096
3. 用户在Apollo操作了一次配置发布，由于Tomcat的类加载机制，会导致打开本地200多个文件，导致超出了4096上限
4. Jedis在运行过程中需要和Redis重新建立连接，然而由于文件句柄数已经超出上限，所以连接失败
5. 应用对外的服务由于无法连接Redis，导致请求超时，客户端请求堆积，陷入恶性循环

## 6.2 后续优化措施
通过这次问题排查，我们不仅对`Too many open files`这一问题有了更深的认识，同时也简单总结下未来可以优化的地方：

1. **操作系统配置**

   从这次的例子可以看出，我们不仅要关心应用内部，对系统的一些设置也需要保持一定的关注。如这次的Max Open Files配置，对于普通应用，如果流量不大的话，使用4096估计也OK。但是对于对外提供服务的应用，4096就显得太小了。
   
2. **应用监控、报警**

	对应用的监控、报警还得继续跟上。比如是否以后可以增加对应用的连接数指标进行监控，一旦超过一定的阈值，就报警。从而可以避免突然系统出现问题，陷于被动。

3. **中间件客户端及早初始化**

	鉴于Tomcat的类加载机制，中间件客户端应该在程序启动的时候做好初始化动作，同时把所有的类都加载一遍，从而避免后续会产生这类诡异的问题。

4. **遇到故障，不要慌张，保留现场**
	
	生产环境遇到故障，不要慌张，如果一时无法解决问题的话，可以通过重启解决。不过应该至少保留一台有问题的机器，从而为后面排查问题提供有利线索。
