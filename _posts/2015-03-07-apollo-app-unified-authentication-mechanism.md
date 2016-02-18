---
layout:     post
title:      手机App的统一认证机制
date:       2015-03-07 14:00:00 +0800
summary:    本文主要介绍了大众点评的阿波罗App如何借助oAuth 2.0来对Restful服务请求进行统一认证的机制。
categories:
---

##一、简介
阿波罗手机端是所有销售和销售主管（5000多人）使用的销售终端。销售可以使用阿波罗手机随时随地接收通知、查看业绩、录入拜访、维护客户、方案等信息，大大提高销售的工作效率（日活4000多人）。同时，阿波罗手机端还在公司内率先采用混合模式开发app，在提高开发效率的同时也为其他手机团队积累了框架和经验。

目前阿波罗各个服务提供方都提供了Restful服务供手机端调用，由于Restful接口通常是无状态的，所以我们设计了一种借助oAuth 2.0来对请求进行统一认证的机制。

下文就会对这一机制做一个详细的介绍。

##二、概览
###2.1 名词解释

在详细介绍之前，先来了解下会涉及到的几个名词

* **SSO服务**
	* 公司的统一认证服务
	* 负责对用户身份进行认证
* **oAuth服务**
	* 公司统一的oAuth服务
	* 负责签发、校验accessToken和refreshToken
* **阿波罗oAuth服务**
	* 用于把阿波罗项目和公司oAuth服务对接的服务，主要负责accessToken的获取和刷新
	* 我们单独设立一个服务，而不是把该项职责下放到客户端的原因是：
		* 公司oAuth服务对每个应用会分配一个appId和secret
		* 每次获取/刷新accessToken的时候都需要把appId和secret传给公司oAuth服务
		* 如果职责下放到客户端，就意味着secret也得暴露给客户端，不安全！
* **阿波罗后端服务**
	* 提供Restful服务的各个业务单元的后端，如客户服务，协议服务，团单服务，拜访服务，业绩服务等
* **阿波罗Native**
	* 阿波罗App的Native部分
	* 包括阿波罗的Native代码以及Efte框架
* **阿波罗前端**
	* 阿波罗App的前端部分
	* 主要是各个业务单元部署在手机端的前端代码（Javascript等）
	* 这些代码负责手机端的页面展现以及后端的数据交互
* **授权码**
	* oAuth服务签发的用于获取accessToken的凭证，时效性一般在几分钟
* **accessToken**
	* oAuth服务签发的用于访问服务的凭证，时效性目前配置为1天
* **refreshToken**
	* oAuth服务签发的用户刷新accessToken的凭证，如果连续3天未使用阿波罗App，就会失效

###2.2 认证机制概览
下图为App整体和各项服务之间的依赖关系
![阿波罗App统一认证整体概览](/images/2015-03-07/apollo-app-overview.png)

其中主要涉及以下两个方面，所以后面会通过这两块的流程图来简要介绍阿波罗统一认证机制的实现。

1. App获取accessToken和refreshToken
2. App通过accessToken来调用Restful服务以及通过refreshToken来刷新accessToken

####2.2.1 App获取accessToken和refreshToken
App启动过程中，会去检查本地是否有认证信息存在，如果没有（第一次启动），就会去服务端请求该信息。

下图简要描述了App如何获取到accessToken和refreshToken
![App获取accessToken和refreshToken](/images/2015-03-07/apollo-app-startup.png)

####2.2.2 App调用Restful服务过程
阿波罗App中有很多业务子模块（如业绩、拜访、客户、协议、团单等），每个业务子模块都会有很多和后端服务交互的过程。由于涉及到了敏感的业务数据和身份信息，所以我们必须要有一种统一的机制来对这一过程加以保障。

下图简要描述了页面调用Restful服务过程中的认证部分，主要是阿波罗Native如何统一处理认证相关的工作。
![App调用Restful服务过程](/images/2015-03-07/apollo-app-call-restful.png)

##三、详细步骤介绍
###3.1 App获取accessToken和refreshToken
在App启动过程中，Native代码会去检查本地是否有认证信息存在。如果有，那么就会直接打开主页面。如果没有（第一次启动），就会去服务端请求该信息。

请求认证信息的详细过程如下：

* Native新建一个webview，加载oAuth授权URL
	* 授权URL中会传入appId和callback url，如
		* https://xxx.com/oauth2.0/authorize?`appId`=yyy&`callbackUrl`=http://oauth_callback
	* `appId`是oAuth服务分配给应用的唯一标识符
	* `callBackUrl`是授权成功后的重定向地址，重定向的时候会带上授权码信息。Native代码通过拦截这个callBackUrl来获得授权码
	* *`callBackUrl`需要事先在oAuth服务那里注册，后面每次认证的时候，oAuth服务都会做校验，判断传入的callBackUrl是否和注册的一致。*
* oAuth服务在授权之前会先要求用户登录，所以它会跳转到SSO来做安全认证
* 如果用户已经登陆过，SSO就会跳转回oAuth服务页面。如果没有登陆过，那么就会要求用户登录。
* 登录成功后，oAuth服务会根据appId和用户的身份，签发一个授权码。后面我们会通过这个凭证，appId和secret来获取accessToken。需要注意的是，这个凭证是有有效期的，一般配置在几分钟。
* oAuth服务把callback url拼上授权码做重定向
* Native代码拦截到callback url，解析出授权码。然后调用阿波罗的oAuth服务
* 阿波罗oAuth服务通过授权码，appId和secret，就能从公司oAuth服务获取到accessToken和refreshToken并返回给客户端
* Native代码把这两个token存储于本地数据空间，以备后面使用

###3.2 App调用Restful服务过程
阿波罗App中有很多业务子模块（如业绩、拜访、客户、协议、团单等），每个业务子模块都会有很多和后端服务交互的过程。由于涉及到了敏感的业务数据和身份信息，所以我们必须要有一种统一的机制来对这一过程加以保障。

另外，阿波罗App是一个混合应用，所有的业务代码都是通过Javascript实现的，尽管JS本身有直接请求Restful服务的能力，我们最后还是决定通过Native来调用HTTP服务。主要原因有以下几点：

1. 通过Native调用，可以解决跨域问题
2. 在使用过程中，由于accessToken的时效性（目前配置是1天过期），会涉及到刷新accessToken，而在刷新accessToken的时候，我们需要能阻塞住全局所有的Restful调用，等accessToken刷新完再继续。这个需求通过Native能很简单的实现，而用JS会非常复杂（因为每个页面都是独立的web view）。

App调用Restful服务的详细过程如下：

* 页面发起ajax请求，由于需要调用Native，我们封装了Efte.ajax组件来统一和Native交互，示例代码如下：
{% highlight javascript%}
Efte.ajax({
  method: 'GET',
  url: 'https://xxx.com/customers',
  success: function(data) {
    //handle data
  },
  error: function(status, message) {
    //handle error
  }
});
{% endhighlight %}
* Efte.ajax通过jsBridge来调用Native方法（jsBridge涉及到Efte框架，如果感兴趣可以看一下[Efte介绍](http://nobodyiam.com/2015/06/01/what-is-efte/)）
* Native代码收到请求后，会组装HTTP请求，设置其中的Authorization header的值为accessToken，然后发出请求。示例代码如下：
{% highlight objective-c%}
AFHTTPClient *client = [[AFHTTPClient alloc] initWithBaseURL:url];
[client setDefaultHeader:@"Authorization" value:[EFTEOAuthViewController accessToken]];
NSMutableURLRequest *request = [client requestWithMethod:method path:urlString parameters:params];
{% endhighlight %}
* 我们要求每个阿波罗后端服务都写一个filter来对请求做校验，不过校验逻辑很简单。首先取出请求中的Authorization信息，然后调用oAuth服务来校验。如果成功，就继续处理请求，如果失败，直接返回HTTP状态码401。示例代码如下：
{% highlight java%}
@Override
public void doFilter(ServletRequest req, ServletResponse resp,
					 FilterChain filterChain) throws IOException, ServletException {
    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) resp;

    String authToken = request.getHeader("Authorization");
    if(!oauthUtil.validate(authToken)) {
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");
        return;
    }
    
    filterChain.doFilter(req, resp);
}
{% endhighlight %}
* Native端判断服务返回是否401，如果不是401，就把结果返回给JS端处理。
* 如果返回的是401，就说明accessToken过期了，执行刷新accessToken逻辑
* 首先启动全局阻塞Restful调用逻辑，把当前请求以及后续的所有Restful调用都存入队列等待。同时之前已经发出的请求由于accessToken过期也会陆续的回来，也会被放入队列等待。
* 从本地取出refreshToken，然后调用阿波罗oAuth服务
* 阿波罗oAuth服务通过refreshToken，appId和secret，调用公司oAuth服务来刷新accessToken
* 如果刷新成功的话，Native端把新的accessToken更新到本地数据空间，解除全局阻塞Restful调用逻辑，同时对队列中的所有请求重发
* 如果刷新失败的话，就说明refreshToken也过期了（连续3天未使用），需要重新进行oAuth授权。这一过程和3.1节的[**App获取accessToken和refreshToken**](#3.1-app获取accesstoken和refreshtoken)过程是一样的，在此就不在赘述了。

#四、小结

通过这样一套机制，我们解决了阿波罗App的安全认证问题，同时还收获了以下好处：

* 安全认证统一实现，维护性和扩展性较好
	* 整个认证逻辑基本都在Native统一实现，所以后续不管是做维护还是扩展新特性都会比较容易
* 对业务基本无侵入，各业务单元接入成本很低
	* 安全认证对前端JS透明，完全无感知
	* 后端接入很轻量，只需实现一个简单的filter逻辑即可
* 借助oAuth 2.0，安全性有保障
