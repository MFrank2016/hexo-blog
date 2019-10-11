---
title: 【Mysql】如果在Mysql中插入1000万条数据
date: 2019-06-04 09:37:41
tags:
 - mysql
 - 测试
categorys:
 - 数据库
---

阅读本文大约需要：5分钟，本文口感：冰镇🍉

## 说明

看到这个标题，也许你的第一反应会是：？？？，莫名其妙插一千万条数据干嘛？嫌mysql空间太大？

![](https://i.loli.net/2019/05/18/5cdf6539270f631562.jpg)

莫方，自有妙用。想想看，如果你有一个上亿的项目，emmmm，别误会，我是说有可能会发展到上亿数据的项目，比如活动的领取记录表，虽然每次活动可能只有几万甚至几十万条数据，但是如果这个活动系统中创建的活动很多，一个月的数据增长量也许会达到百万甚至千万级别，一年下来，数据量可能就真上亿了。

对于Mysql数据库来说，当数据量达到千万的量级时，可能就会发生一些变化，比如某些sql可能就会因为没有命中索引而十分缓慢，在一个上千万数据的表中进行非最左前缀模糊匹配时，你就会知道什么叫做绝望。所以初期看起来运行良好的程序，等到数据量增大到一定程度时，就容易变得十分缓慢。所以设计一个比较重要的数据表时，用假数据来跑一跑看sql的执行效率是个不错的选择。

于是便有了本篇的话题，如何往Mysql中插入一千万条数据？

## 初期准备

Mysql版本为8.0.11：

![](https://i.loli.net/2019/06/05/5cf71abecc5be80960.png)

默认存储引擎为InnoDB：

![](https://i.loli.net/2019/06/05/5cf71e71d75b790847.png)

先创建一个测试用的数据库以及测试用的一张表：

![](https://i.loli.net/2019/06/05/5cf71c9e7335f90543.png)

```sql
create database if not exists test_db default charset utf8 collate utf8_general_ci;
```

上面这句就是创建数据库的命令，`if not exists` 代表如果不存在则创建，如果已存在则不执行。`default charset utf8 collate utf8_general_ci` 则为设置数据库的默认字符集为`utf8`，设置默认的字符比较规则为`utf8_general_ci`。

说到字符集和比较规则，顺便提一句，每种字符集对应若干种比较规则，每种字符集都有默认的比较规则，`utf8_general_ci`是`utf8`的默认比较规则，`_ci`结尾代表这种比较规则不区分大小写。

接下来创建测试用的表：

![](https://i.loli.net/2019/06/10/5cfdb4f13ab8761814.png)

```sql
create table `order_detail` (
    `id` bigint(11) not null auto_increment,
    `order_no` varchar(50) not null comment '订单号',
    `commodity_id` bigint(11) not null comment '商品id',
    `amount` decimal(20,6) not null comment '订单金额',
    `buyer_id` bigint(11) not null comment '付款人id',
    `status` int(2) not null comment '订单状态',
    `created_time` datetime not null comment '创建时间',
    `updated_time` datetime not null comment '更新时间', 
    primary key(`id`)
) engine InnoDB charset utf8mb4 collate utf8mb4_unicode_ci;
```

这里先不加索引，后续会比较有索引和没有索引的情况下，插入数据的效率。

## 暴力插入法

不管是啥，盘它就是了，撸起袖子就是干。新建一个 `springboot` 项目，