---
layout:     post
title:      MySQL支持emoji字符
date:       2016-05-29 14:00:00 +0800
summary:    最近在项目中需要能够支持保存emoji字符（如🚀），第一反应是使用utf8mb4编码，不过在具体实施过程中还是发现有一些细节是需要特别注意的，所以在此记录一下。
categories:
---
最近在项目中需要支持用户输入emoji字符（如🚀），由于数据库的编码用的是utf8，所以需要做一些改造来予以支持。

## 1. 背景知识
- `UTF8`
	- 在MySQL中<a href="http://dev.mysql.com/doc/refman/5.7/en/charset-unicode-utf8.html" target="_blank">utf8</a>编码的字符会占用1-3个字节来存储数据
	- 只支持<a href="https://en.wikipedia.org/wiki/Plane_(Unicode)#Basic_Multilingual_Plane" target="_blank">BMP</a>字符
- `UTF8MB4`
	- 在MySQL中<a href="http://dev.mysql.com/doc/refman/5.7/en/charset-unicode-utf8mb4.html" target="_blank">utf8mb4</a>编码的字符会占用1-4个字节来存储数据
	- 支持<a href="https://en.wikipedia.org/wiki/Plane_(Unicode)#Basic_Multilingual_Plane" target="_blank">BMP</a>字符
	- 还支持<a href="https://en.wikipedia.org/wiki/Plane_(Unicode)#Supplementary_Multilingual_Plane" target="_blank">Supplementary</a>字符，比如emoji符号
	- MySQL 5.5以上才有utf8mb4编码
	
所以，从上面可以看出，utf8mb4是utf8的超集，所以从utf8升级到utf8mb4，理论上是没有任何问题的，不过对数据库版本有要求（5.5以上）。


## 2. 改造步骤

### 2.1 MySQL服务端

#### 2.1.1 修改/etc/my.cnf并重启MySQL服务器

这一步非必须，如果实在由于某些原因无法修改数据库配置，可以通过后面`2.3.3`部分的客户端手动配置来解决。

{% highlight properties%}
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_general_ci
{% endhighlight %}


### 2.2 MySQL数据库

#### 2.2.1 修改数据库编码为utf8mb4

{% highlight sql%}
ALTER DATABASE `DBName` CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci;
{% endhighlight %}

可以通过以下SQL来生成alter语句：

{% highlight sql%}
use information_schema;
SELECT concat("ALTER DATABASE `",table_schema,"` CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci;") as _sql
FROM `TABLES` where table_schema like "yourDbName" group by table_schema;
{% endhighlight %}

#### 2.2.2 修改数据表编码为utf8mb4

{% highlight sql%}
ALTER TABLE `TableName` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
{% endhighlight %}

可以通过以下SQL来生成所有表的alter语句：

{% highlight sql%}
use information_schema;
SELECT concat("ALTER TABLE `",table_schema,"`.`",table_name,"` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;") as _sql
FROM `TABLES` where table_schema like "yourDbName" group by table_schema, table_name;
{% endhighlight %}

#### 2.2.3 修改表字段编码为utf8mb4

如果表字段没有特别指定编码的，就会默认使用表的编码，所以这步一般都可以省略。不过如果有不一样的话，可以通过下面SQL来修改。

{% highlight sql%}
ALTER TABLE `TableName ` CHANGE `ColumnName ` `ColumnName` `DataType(Length)` CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT `xx` Comment `xx`;
{% endhighlight %}

### 2.3 Java客户端

#### 2.3.1 升级JDBC驱动

确保mysql connector版本高于5.1.13，参见<a href="http://dev.mysql.com/doc/relnotes/connector-j/5.1/en/news-5-1-13.html" target="_blank">release note</a>。

#### 2.3.2 JDBC数据库连接串
由于目前的java mysql connector还不支持`utf8mb4`，所以在jdbc的数据库连接串还只能使用utf8

>jdbc:mysql://database_ip:3306/database_name?characterEncoding=utf8

#### 2.3.3 数据库连接配置

如果在2.1部分，已经修改了数据库的配置并重启，那么这一步可以省略，因为mysql connector会自动继承服务器的设置。

如果2.1部分由于各种原因没有修改的话，那么就需要在客户端这里做设置了。

具体来说，就是在每个数据库连接建立的时候，通过运行`set names utf8mb4`来显式设置客户端连接的encoding为`utf8mb4`。

如果客户端用的是<a href="https://tomcat.apache.org/tomcat-7.0-doc/jdbc-pool.html" target="_blank">tomcat-jdbc</a>的话，可以通过`initSQL`属性来做设置。

> initSQL - the ability to run a SQL statement exactly once, when the connection is created

对spring-boot应用而言，就是设置`spring.datasource.initSQL`属性：

> spring.datasource.initSQL=set names utf8mb4

## 3. 使用utf8mb4后需要注意的地方

由于utf8mb4的字节长度是1-4个字节，而utf8的字节长度是1-3个字节，所以会有一些限制的变化。

### 3.1 字段长度限制

MySQL中的字段类型都有长度限制，比如`varchar`的最长字节长度是65535，所以使用utf8编码的时候，可以指定字段最长为65535/3=21845。如果使用utf8mb4编码的话，由于字符最长会占用4个字节，所以字段最长只能为65535/4=16383。

### 3.2 索引长度限制

MySQL中的索引也有长度限制： 767字节，所以使用utf8编码的的时候，可以指定索引字段最长为255字节，但是指定utf8mb4的话，只能索引191字节。

> KEY `XX_Index` (`XX`(191))

### 3.3 行长度限制

MySQL中的行也有长度限制： 65535字节，所以当字段编码从utf8变为utf8mb4的时候，可能也会需要缩短部分字段的长度来满足行的长度限制。

## 4. 总结

从上面的改造步骤来看，从utf8到utf8mb4还是比较容易的，而且理论上来说是没有风险的，所以推荐大家尽快采用utf8mb4。另外，对于新的应用可以一上来就使用utf8mb4编码，以免真的碰到问题了再来改造过于被动。