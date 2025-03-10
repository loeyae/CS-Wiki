# 🚀 NoSQL概述

---

## 1. 为什么要用 NoSQL

**1）单机 MySQL 的年代**

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20200714152020.png" style="zoom: 80%;" />

90年代，一个基本的网站访问量一般不会太大，单个数据库完全足够。那个时候，更多的去使用静态网页 Html，服务器根本没有太大的压力。

❓ <u>思考一下，这种情况下：整个网站的瓶颈是什么？</u>

- 数据量如果太大、一个机器放不下
- 数据的索引 （B+ Tree），一个机器内存也放不下
- 访问量（读写混合），一个服务器承受不了

**2）Memcached（缓存） + MySQL + 垂直拆分 （读写分离）**

网站 80% 的情况都是在读，每次都要去查询数据库的话就十分的麻烦。所以说我们希望减轻数据的压力，我们可以使用缓存来保证效率。

发展过程： 优化数据结构和索引 --> 文件缓存（IO）--> Memcached（当时最热门的技术）

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20200714152426.png)

**3）分库分表 + 水平拆分 + MySQL 集群**

技术和业务在发展的同时，对人的要求也越来越高

本质：数据库（读，写）

早些年 **MyISAM： 表锁**，十分影响效率！高并发下就会出现严重的锁问题

转战 **Innodb：行锁**

慢慢的就开始使用分库分表来解决写的压力。 MySQL 推出了表分区，然而并没有多少公司使用。MySQL 的集群，很好满足了那个年代的所有需求。

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20200714152744.png)

**4）如今最近的年代**

2010--2020 十年之间，世界已经发生了翻天覆地的变化。用户的个人信息，社交网络，地理位置。用户自己产生的数据，用户日志等等爆发式增长，对于 MySQL 等关系型数据库来说，一旦数据库表很大，效率就低了。而且在大数据的 IO  压力下，表已经几乎没法更大了。

这时候我们就需要使用 NoSQL 数据库，NoSQL 可以很好的处理以上的情况 👇

## 2. 什么是 NoSQL

**NoSQL = Not Only SQL （不仅仅是SQL）**

**NoSQL 用来泛指非关系型数据库，采用 `Map<String,Object>` 键值对来存储数据**

> 关系型数据库：表格 ，行 ，列

随着 web2.0 互联网的诞生，传统的关系型数据库很难对付 web2.0 时代，尤其是超大规模的高并发的社区，暴露出来很多难以克服的问题。NoSQL在当今大数据环境下发展的十分迅速，Redis 是发展最快的，而且是我们当下必须要掌握的一个技术。很多的数据类型用户的个人信息，社交网络，地理位置。这些数据类型的存储不需要一个固定的格式，不需要多余的操作就可以横向扩展 。

> 💡 **大数据时代的 3 V**：主要是描述问题的
> - 海量 Volume
> - 多样 Variety
> - 实时 Velocity
> 
> **大数据时代的 3 高**：主要是对程序的要求
>
> - 高并发
> - 高可扩
> - 高性能

🎯 **NoSQL 的特点如下**：（**解耦**）

- 方便扩展（数据之间没有关系，很好扩展）
- 大数据量高性能（Redis 一秒写8万次，读取11万，NoSQL的缓存记录级，是一种细粒度的缓存，性
  能会比较高）
- 数据类型是多样型的（不需要事先设计数据库，随取随用）

🌂 **对比传统 RDBMS 和 NoSQL：**

## 3. 阿里巴巴演进分析

❓ 对于淘宝上这么多商品信息难道都是在一个数据库中的吗? 显然不是：

- **商品的基本信息**

  名称、价格、商家信息；

  关系型数据库就可以解决了！ MySQL / Oracle 

  （淘宝内部的 MySQL 不是大家用的 MySQL）

- **商品的描述、评论（文字比较多）**

  文档型数据库中，MongoDB

- **图片**

  分布式文件系统 FastDFS

  - 淘宝自己的 TFS
  - Gooale的 GFS
  - Hadoop HDFS
  - 阿里云的 oss

- **商品的关键字 （搜索）**
  - 搜索引擎 solr、elasticsearch
  - ISerach：多隆

- **商品热门的波段信息**
  - 内存数据库
  - Redis、Tair、Memache...

- **商品的交易，外部的支付接口**
  
  - 三方应用

## 4. NoSQL 的四大分类

① **KV 键值对**：

- 新浪：**Redis**
- 美团：Redis + Tair
- 阿里、百度：Redis + memecache

② **文档型数据库（bson 格式和 json一样）**：

- **MongoDB** （一般必须要掌握）
  - MongoDB 是一个基于分布式文件存储的数据库，C++ 编写，主要用来处理大量的文档！
  - MongoDB 是一个介于关系型数据库和非关系型数据中中间的产品！MongoDB 是非关系型数
    据库中功能最丰富，最像关系型数据库的！
  
- ConthDB
  

③ **列存储数据库**

- **HBase**
- 分布式文件系统

④ **图形 (Graph) 数据库**

- 他不是存图形，放的是关系，比如：朋友圈社交网络，广告推荐
- **Neo4j**，InfoGrid；

🍹 四者对比：

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20200714154901.png)

## 📚 References

- [【狂神说Java】Redis最新超详细版教程通俗易懂](https://www.bilibili.com/video/BV1S54y1R7SB?from=search&seid=3325634079268895938)