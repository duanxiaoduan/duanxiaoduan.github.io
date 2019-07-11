---
layout:     post
title:      MySQL存储引擎MyISAM和InnoDB底层索引结构
subtitle:   mysql
date:       2019-07-11
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - mysql
---

MySQL存储引擎MyISAM和InnoDB底层索引结构
============================

**目录**

[一 存储引擎作用于什么对象](#%E4%B8%80%20%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%E4%BD%9C%E7%94%A8%E4%BA%8E%E4%BB%80%E4%B9%88%E5%AF%B9%E8%B1%A1)

[二 MyISAM和InnoDB对索引和数据的存储在磁盘上是如何体现的](#%E4%BA%8C%20MyISAM%E5%92%8CInnoDB%E5%AF%B9%E7%B4%A2%E5%BC%95%E5%92%8C%E6%95%B0%E6%8D%AE%E7%9A%84%E5%AD%98%E5%82%A8%E5%9C%A8%E7%A3%81%E7%9B%98%E4%B8%8A%E6%98%AF%E5%A6%82%E4%BD%95%E4%BD%93%E7%8E%B0%E7%9A%84)

[三 MyISAM主键索引与辅助索引的结构](#%E4%B8%89%20MyISAM%E4%B8%BB%E9%94%AE%E7%B4%A2%E5%BC%95%E4%B8%8E%E8%BE%85%E5%8A%A9%E7%B4%A2%E5%BC%95%E7%9A%84%E7%BB%93%E6%9E%84)

[1\. 主键索引：](#1.%20%E4%B8%BB%E9%94%AE%E7%B4%A2%E5%BC%95%EF%BC%9A)

[2\. 辅助（非主键）索引：](#2.%20%E8%BE%85%E5%8A%A9%EF%BC%88%E9%9D%9E%E4%B8%BB%E9%94%AE%EF%BC%89%E7%B4%A2%E5%BC%95%EF%BC%9A)

[四 InnoDB主键索引与辅助索引的结构](#%E5%9B%9B%20InnoDB%E4%B8%BB%E9%94%AE%E7%B4%A2%E5%BC%95%E4%B8%8E%E8%BE%85%E5%8A%A9%E7%B4%A2%E5%BC%95%E7%9A%84%E7%BB%93%E6%9E%84)

[1\. 主键索引：](#1.%20%E4%B8%BB%E9%94%AE%E7%B4%A2%E5%BC%95%EF%BC%9A)

[2\. 辅助（非主键）索引：](#2.%20%E8%BE%85%E5%8A%A9%EF%BC%88%E9%9D%9E%E4%B8%BB%E9%94%AE%EF%BC%89%E7%B4%A2%E5%BC%95%EF%BC%9A)

[五 InnoDB索引结构需要注意的点](#%E4%BA%94%20InnoDB%E7%B4%A2%E5%BC%95%E7%BB%93%E6%9E%84%E9%9C%80%E8%A6%81%E6%B3%A8%E6%84%8F%E7%9A%84%E7%82%B9)

* * *

**PS：为了更好地理解本文内容，我强烈建议先阅读完我的上一篇文章[深入理解MySQL索引底层数据结构与算法](https://blog.csdn.net/u010922732/article/details/82992920)**

一 存储引擎作用于什么对象
=============

存储引擎是作用在表上的，而不是数据库。

二 MyISAM和InnoDB对索引和数据的存储在磁盘上是如何体现的
==================================

先来看下面创建的两张表信息，role表使用的存储引擎是MyISAM，而user使用的是InnoDB：

![](https://img-blog.csdn.net/20181012092653724?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjI3MzI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

再来看下两张表在磁盘中的索引文件和数据文件：

![](https://img-blog.csdn.net/20181012092702772?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjI3MzI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**1\. role表有三个文件，对应如下：**

*   role.frm：表结构文件
*   role.MYD：数据文件（MyISAM Data）
*   role.MYI：索引文件（MyISAM Index）

**2\. user表有两个文件，对应如下：**

*   user.frm：表结构文件
*   user.ibd：索引和数据文件（InnoDB Data）

也由于两种引擎对索引和数据的存储方式的不同，我们也称MyISAM的索引为非聚集索引，InnoDB的索引为聚集索引。

三 MyISAM主键索引与辅助索引的结构
====================

我们先列举一部分数据出来分析，如下：

![](https://img-blog.csdn.net/20181012092712183?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjI3MzI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

上面已经说明了MyISAM引擎的索引文件和数据文件是分离的，我们接着看一下下面两种索引结构异同。

### **1\. 主键索引：**

上一篇文章已经介绍过数据库索引是采用B+Tree存储，并且只在叶子节点存储数据，在MyISAM引擎中叶子结点存储的数据其实是索引和数据的文件指针两类。

如下图中我们以Col1列作为主键建立索引，对应的叶子结点储存形式可以看一下表格。

![](https://img-blog.csdn.net/20181012092744524?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjI3MzI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

索引值

15

18

→

20

30

→

49

50

→

文件指针

0x07

0x56

0x6A

0xF3

0x90

0x77

通过索引查找数据的流程：先从索引文件中查找到索引节点，从中拿到数据的文件指针，再到数据文件中通过文件指针定位了具体的数据。

### **2\. 辅助（非主键）索引：**

以Col2列建立索引，得到的辅助索引结构跟上面的主键索引的结构是相同的。

![](https://img-blog.csdn.net/20181012092806447?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjI3MzI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

四 InnoDB主键索引与辅助索引的结构
====================

### 1\. 主键索引：

我们已经知道InnoDB索引是聚集索引，它的索引和数据是存入同一个.idb文件中的，因此它的索引结构是在同一个树节点中同时存放索引和数据，如下图中最底层的叶子节点有三行数据，对应于数据表中的Col1、Col2、Col3数据项。

![](https://img-blog.csdn.net/2018101209282660?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjI3MzI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 2\. 辅助（非主键）索引：

这次我们以数据表中的Col3列的字符串数据建立辅助索引，它的索引结构跟主键索引的结构有很大差别，我们来看下面的图：

在最底层的叶子结点有两行数据，第一行的字符串是辅助索引，按照ASCII码进行排序，第二行的整数是主键的值。

![](https://img-blog.csdn.net/20181012092836770?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjI3MzI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

五 InnoDB索引结构需要注意的点
==================

1\. 数据文件本身就是索引文件

2\. 表数据文件本身就是按B+Tree组织的一个索引结构文件

3\. 聚集索引中叶节点包含了完整的数据记录

4\. InnoDB表必须要有主键，并且推荐使用整型自增主键

正如我们上面介绍InnoDB存储结构，索引与数据是共同存储的，不管是主键索引还是辅助索引，在查找时都是通过先查找到索引节点才能拿到相对应的数据，如果我们在设计表结构时没有显式指定索引列的话，MySQL会从表中选择数据不重复的列建立索引，如果没有符合的列，则MySQL自动为InnoDB表生成一个隐含字段作为主键，并且这个字段长度为6个字节，类型为整型。

那为什么推荐使用整型自增主键而不是选择UUID？

*   UUID是字符串，比整型消耗更多的存储空间；
*   在B+树中进行查找时需要跟经过的节点值比较大小，整型数据的比较运算比字符串更快速；
*   自增的整型索引在磁盘中会连续存储，在读取一页数据时也是连续；UUID是随机产生的，读取的上下两行数据存储是分散的，不适合执行where id > 5 && id < 20的条件查询语句。
*   在插入或删除数据时，整型自增主键会在叶子结点的末尾建立新的叶子节点，不会破坏左侧子树的结构；UUID主键很容易出现这样的情况，B+树为了维持自身的特性，有可能会进行结构的重构，消耗更多的时间。
*   为什么非主键索引结构叶子节点存储的是主键值？

保证数据一致性和节省存储空间，可以这么理解：商城系统订单表会存储一个用户ID作为关联外键，而不推荐存储完整的用户信息，因为当我们用户表中的信息（真实名称、手机号、收货地址···）修改后，不需要再次维护订单表的用户数据，同时也节省了存储空间。
