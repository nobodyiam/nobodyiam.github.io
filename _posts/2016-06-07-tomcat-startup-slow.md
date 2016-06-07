---
layout:     post
title:      Tomcat启动缓慢问题解决
date:       2016-06-07 22:00:00 +0800
summary:    最近在项目中发现有时候Tomcat启动非常慢，本文简单描述了其中一个比较大的原因和解决方案。
categories:
---
最近在项目中发现有时候Tomcat启动非常慢，有时候要花上好几分钟启动。

一开始以为是Spring Boot的AutoConfiguration导致的，不过后来仔细看了启动日志后发现罪魁祸首是这个：

> Creation of SecureRandom instance for session ID generation using [SHA1PRNG] took **[100,138]** milliseconds.

这个`SecureRandom`的初始化竟然花了100秒之多。。。

后来查了一下，发现这个问题抱怨的还是蛮多的，以至于tomcat的<a href="https://wiki.apache.org/tomcat/HowTo/FasterStartUp#Entropy_Source" target="_blank">wiki</a>里面还单独列出来作为加速启动的一个方面：

*Tomcat 7+ heavily relies on SecureRandom class to provide random values for its session ids and in other places. Depending on your JRE it can cause delays during startup if entropy source that is used to initialize SecureRandom is short of entropy. You will see warning in the logs when this happens, e.g.:*
	
	<DATE> org.apache.catalina.util.SessionIdGenerator createSecureRandom
	INFO: Creation of SecureRandom instance for session ID generation using [SHA1PRNG] took [5172] milliseconds.
	
*There is a way to configure JRE to use a non-blocking entropy source by setting the following system property: `-Djava.security.egd=file:/dev/./urandom`*

尝试使用`-Djava.security.egd=file:/dev/./urandom`启动了一下，果然快了很多。

不过tomcat的wiki中提到，如果使用这个非阻塞的`/dev/urandom`的话，会有一些安全方面的风险，这块我倒确实不太明白，不过好在有明白人，而且还写了一篇<a href="http://www.2uo.de/myths-about-urandom/" target="_blank">长文</a>来证明使用`/dev/urandom`是没问题的，所以就先用着吧：-）
