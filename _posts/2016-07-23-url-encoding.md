---
layout:     post
title:      URL Encoding二三事
date:       2016-07-23 22:00:00 +0800
summary:    URL Encoding是Web编程中比较常见而且基础的知识，不过似乎也很容易犯错，本文旨在稍微系统的介绍下关于URL和URL Encoding的一些知识，从而希望对大家有一些启发。
categories:
---
URL Encoding是Web编程中比较常见而且基础的知识，不过似乎也很容易犯错，本文旨在稍微系统的介绍下关于URL和URL Encoding的一些知识，从而希望对大家有一些启发。

## URL简介
在这个网络非常发达的年代，相信大家对URL都是非常之熟悉，日常在浏览器中输入的就是URL，比如<a href="http://nobodyiam.com" target="_blank">http://nobodyiam.com</a>就是一个非常典型的URL。

### URL的组成部分
一个完整的URL分为很多个组成部分，如下图所示：

    http://example.com:8042/over/there?name=ferret#nose
    \__/   \______________/\_________/ \_________/ \__/
      |           |            |            |        |
    scheme    authority       path        query   fragment

* Scheme
	* 协议部分，常见的有http, https, ftp等
* Authority
	* 这部分较常见的就是一个域名，如www.google.com
	* 不那么常见的还会带上用户名、密码和端口信息
* Path
	* 路径部分，通常会有层次结构，如/customers/100/orders
* Query
	* 参数部分，如搜索功能常用的keyword=xxx
* Fragment
	* 这部分就是我们常说的hash部分，一般用来做页面内的定位
	* 不过在一些前端框架中会被用来作为页面跳转的定位

## 为什么URL需要做Encoding
大致了解了一下URL之后，大家可能会问，我们为啥需要对好端端的URL做encoding呢？

其实答案很简单，因为根据<a href="https://tools.ietf.org/html/rfc3986#section-2.2" target="_blank">rfc3986</a>规范所定义的，在URL中有一些字符是有保留字符，他们在URL中有着特殊含义，所以如果保留字符在URL中被用做其它用途的话，就必须使用<a href="https://tools.ietf.org/html/rfc3986#section-2.1" target="_blank">Percent-Encoding</a>做encode。

>URI producing applications should percent-encode data octets that correspond to characters in the reserved set unless these characters are specifically allowed by the URI scheme to represent data in that component.

## 对URL做Encoding
那现在我们来看看该如何对URL做encoding。其实最专业的方式当然是熟读`rfc3986`，然后按照规范对各种不同场景下的保留字符或不符合规范的字符使用`Percent-Encoding`编码了。

不过这么做有点太累，而且事实上各种类库都已经提供了支持，所以我们直接使用即可。

### 使用Guava做Url Encoding
<a href="https://github.com/google/guava" target="_blank">Guava</a>作为Java项目几乎必备的类库，自然也是提供了很好的支持。

下面以`http://example.com/{path1}/{path2}?keyword={someKeyword}#{someFragment}`为例，看看如何使用Guava提供的<a href="https://google.github.io/guava/releases/19.0/api/docs/com/google/common/net/UrlEscapers.html" target="_blank">UrlEscapers</a>来完成这项任务。

UrlEscapers提供了3个静态方法来分别获得对应URL不同部分的escaper：

* public static Escaper urlFormParameterEscaper()
	* 用来对query或者form提交的参数做escaping
* public static Escaper urlPathSegmentEscaper()
	* 用来对path部分做escaping
* public static Escaper urlFragmentEscaper()
	* 用来对fragment部分做escaping

有了这几个方法，就非常容易了，参考下面代码示例。

<script src="https://gist.github.com/nobodyiam/420b9fab2c2f6d477ae1b73400961506.js"></script>

最后的结果就是

>http://example.com/somePathWithSpace%20/somePathWith%E4%B8%AD%E6%96%87?keyword=someKeywordWith%E4%B8%AD%E6%96%87AndPlus%2B#someFragmentWithSpace%20

## 小结
本文样例使用了Guava实现Url Encoding，其实这类类库还有很多，比如Spring的<a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/util/UriUtils.html" target="_blank">UriUtils</a>等都能很好的完成这一工作。

另外如果使用一些封装比较好的工具如<a href="http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html" target="_blank">RestTemplate</a>的话，只要按照它的规范使用，在做Http请求的时候甚至都不需要我们自己做Url Encoding了。

不过万变不离其宗，我们程序员不管使用哪种工具、哪种语言，都应该了解一下它背后的原理，这样万一碰到了诡异的问题，也能够从容不迫的予以解决，所以也希望本文在Url Encoding方面能给大家一些启发吧。
