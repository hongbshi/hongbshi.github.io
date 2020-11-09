---
title: InnoDB Data Dictionary
date: 2020-11-09 20:30:35
categories:
- Mysql
tags:
- Mysql
- InnoDB
---

## 一. 基础知识
1. 本文使用的Mysql版本: `8.0.12-debug`
2. 本文用到的Linux命令: `xxd`
3. 本文需要使用到的知识点: `Mysql` 数据页存储, 可以参见 `https://segmentfault.com/a/1190000037436803`

### 1.1 问题
1. Data Dictironary是什么?
- Data Dictironary(DD, 数据字典)是有关数据库对象的合集, 例如表、视图、索引等, 可以看做是数据库的元信息。换句话说, 数据字典存储了有关表结构的信息, 每个表具有的列, 表的索引等。

2. 系统表是什么? 跟自己创建的表有何不同?
- 系统表有很多, 常见的有`mysql.schemata`,`mysql.tables`, `mysql.indexes`
- 我们创建的表的元信息是放到系统表中的
- 在内存中, 这些元信息以对象的方式提供给外部使用, 比如说, 创建一个表, 内存中会创建这个表的数据字典对象, 系统表以及我们创建的表都会有自己的数据字典对象
- 可以认为系统表的元信息就存储在自己的数据字典对象中, 这些信息会被序列化到磁盘的mysql.ibd文件中

3. DD存储在哪些地方?

这里是针对Mysql 8的数据字典,
- 数据字典信息需要持久化, 存储在mysql.ibd文件中, 使用专门的表空间id
- 在每个独立的表空间中, 也备份了一份这个表空间相关的dd对象序列化信息
- 系统表空间, 也就是ibdata1文件中, 并没有存储dd对象的相关信息

### 1.2 DD存储
Mysql 8之前, 数据字典存储结构如下图,

![数据字典](https://picturestore.nos-eastchina1.126.net/mysql/%E6%95%B0%E6%8D%AE%E5%AD%97%E5%85%B81.jpg)

从图中可以看出, DD信息存储在多个地方,
- 部分数据存储在文件中
- 部分数据存储在mysql系统表中
- InnoDB系统表也存储了部分数据

这种存储方式存在以下问题:
1. 数据存储在多个地方, 难以维护管理
2. MyISAM系统表容易损坏
3. 不支持原子操作

Mysql 8 对数据字典进行了重新设计, 并且使用InnoDB存储,

![数据字典-Mysql 8](https://picturestore.nos-eastchina1.126.net/mysql/%E6%95%B0%E6%8D%AE%E5%AD%97%E5%85%B82.jpg)

- 将原来存储数据字典的文件全部删除, 统一放到数据字典表空间中
- 对于原来使用MyISAM存储的系统表, 全部替换为InnoDB存储, 为实现原子DDL提供了可能性

### 1.3 如何查看数据字典
1. 通过命令行连接Mysql服务器查看
```
SET SESSION debug='+d,skip_dd_table_access_check';
SELECT name, schema_id, hidden, type FROM mysql.tables where schema_id=1 AND hidden='System';
```

可以看到DD中存储了很多系统表,
```
+------------------------------+-----------+--------+------------+
| name                         | schema_id | hidden | type       |
+------------------------------+-----------+--------+------------+
| catalogs                     |         1 | System | BASE TABLE |
| character_sets               |         1 | System | BASE TABLE |
| collations                   |         1 | System | BASE TABLE |
| columns                      |         1 | System | BASE TABLE |
| dd_properties                |         1 | System | BASE TABLE |
| events                       |         1 | System | BASE TABLE |
| index_stats                  |         1 | System | BASE TABLE |
| indexes                      |         1 | System | BASE TABLE |
| schemata                     |         1 | System | BASE TABLE |
| tables                       |         1 | System | BASE TABLE |
| tablespace_files             |         1 | System | BASE TABLE |
| tablespaces                  |         1 | System | BASE TABLE |
+------------------------------+-----------+--------+------------+
```
- 这里我们仅列出了一部分, InnoDB还存储了很多其他的系统表
- 这些系统表存储了各种元数据, columns存储了表的列信息, indexes则存储了表的索引信息, schemata存储了数据库的信息。

2. 通过 `ibd2sdi` 工具查看
```
ibd2sdi test.ibd
```
- tips: 可以通过utilities/ibd2sdi.cc文件, 查看反序列化过程, 进而可以看出sdi页面存储结构

### 1.4 Serialized Dictionary Information(SDI)
1. SDI是什么?
- SDI, 字典序列化信息, 也就是将数据字典中的对象序列化后的数据

2. SDI如何组织?
- DD中包含很多的数据字典对象, 这些对象的SDI数据以B+树的方式进行组织
- SDI数据基本都是压缩后存储的, 压缩方式使用的是zlib

3. SDI举例

```json
{
    "mysqld_version_id":80012,
    "dd_version":80012,
    "sdi_version":1,
    "dd_object_type":"Table",
    "dd_object":{
        "name":"x",
        "columns":[
            {
                "name":"id",
                "type":4,
                ...
            },
            {
                "name":"DB_TRX_ID",
                "type":10,
                ...
            },
            {
                "name":"DB_ROLL_PTR",
                "type":9,
                ...
            }
        ],
        ...
        "indexes":[
            {
                "name":"PRIMARY",
                "hidden":false,
                ...
            }
        ],
        ...
    }
}
```
- 可以看到, SDI中存储了这个表的元信息, 比如说这个表的列组成, 索引信息等。


## 二. SDI存储页示例

上文中, 我们说过在每个表的独立表空间中, 存储了这个表相关的DD对象序列化后的信息。在这一节, 我们学习如何查看SDI信息。

1. 我们首先建立一个表
```
CREATE TABLE `x` (
  `id` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```
- 表所在的库是hbshi

2. 查看这表的独立表空间文件, 通过 `xxd x.ibd x.txt` 查看这个表空间的数据页, 这里我们主要看第4页, 这1页存储了这个表相关的数据字典序列化信息,

```
000c000: 723e 3b3c 0000 0003 ffff ffff ffff ffff  r>;<............
000c010: 0000 0000 0128 06fc 45bd 0000 0000 0000  .....(..E.......
000c020: 0000 0000 0004 0002 04d2 8004 0000 0000  ................
000c030: 0172 0001 0001 0002 0000 0000 0000 0000  .r..............
000c040: 0000 ffff ffff ffff ffff 0000 0004 0000  ................
000c050: 0002 00f2 0000 0004 0000 0002 0032 0100  .............2..
000c060: 0201 0f69 6e66 696d 756d 0003 000b 0000  ...infimum......
000c070: 7375 7072 656d 756d cb80 0000 10ff f100  supremum........
000c080: 0000 0200 0000 0000 0000 0900 0000 0016  ................
........
........
000fff0: 0000 0000 0070 0063 723e 3b3c 0128 06fc  .....p.cr>;<.(..
```

对于一个表空间而言, 可能会有多个数据字典对象, 这些对象序列化后会以B+树的方式组织, 我们看一下具体的存储结构,

整体页存储结构:
- 最小记录: 记录头[00c05e, 00c062], 具体数据[00c063, 00c06a]
- 最大记录: 记录头[00c06b, 00c06f], 具体数据[00c070, 00c077]
- 第1条记录: 2字节的内容长度[00c078, 00c079]; 5字节的记录头[00c07a, 00c07e]; 记录内容[00c07f, 00c16a]。
- 第2条记录: 2字节的内容长度[00c16b, 00c16c], 5字节的记录头[00c16d, 00c171], 记录的数据[00c172, 00c4d1]

我们再看下每个记录的具体存储:
- 长度: 1或者2个字节的长度字段, 说明**内容长度**
- 5个字节的记录头
- 33个字节的记录说明
- 记录的具体内容, 记录头前面的1或者2个字节就是说明这个记录的内容长度的

记录说明:
- 4个字节的类型信息
- 8个字节的id
- 6个字节的事务id
- 7个字节的回滚指针
- 4个字节的非压缩长度
- 4个字节的压缩长度

存储结构如下图所示,
![SDI存储页](https://picturestore.nos-eastchina1.126.net/mysql/SDI%E5%AD%98%E5%82%A8%E9%A1%B5.jpg)

3. SDI记录遍历流程
- 从最小记录计算第1条记录, 00c063 + 010f = 00c172
- 计算第2条记录, 00c172 + ff0d = 00c07f
- 计算第3条记录: 00c07f + fff1 = 00c070

注意: 游标以每个记录的内容开始, 例如最小记录的开始位置为00c063, 而不是00c05e。计算的下一个记录位置, 也是内容开始的位置。

4. 如何查看SDI压缩后的数据
- 首先通过bin/ibd2sdi工具, 将ibd文件中的sdi信息打印
- 从打印中的信息中, 选择其中一项
- 从选中的一项中抽出object字段, 将其压缩, 这里我们使用python脚本进行压缩, 压缩方式使用zlib
```
#!/usr/bin/env python
# -*- coding=utf-8 -*-
#
import os
import sys
import json
import time
import datetime
import hashlib
import zlib

reload(sys)
sys.setdefaultencoding("utf-8")

def help():
    f = open("./test.data")
    l = f.readline()
    d = zlib.compress(l)
    f2 = open("./tmp.data", 'w+')
    f2.write(d)

help()
```
- 对比压缩后的数据以及表空间存储的数据

## 三. DD表空间

1. InnoDB常用表空间
- 表空间占用4个字节, 最大值为0xffff ffff, 这个表空间是无效的, 没有使用
- `0xffff fffe` 是data dicitonary的表空间, 也就是mysql.ibd
- `0xffff fffd` 是临时表空间, 也就是ibtmp1.ibd
- [`0xffff fffc`, `0xffff fff0`]是为日志准备的
- [`0xffff ffef`, `0xffff ff70`]是undo log
- [`0x0000 0002`, `0xffff ff6f`]是可以正常使用的表空间
- 1是 `sys/sys_config` 表空间
- 0是系统表空间, 也就是ibdata1

2. DD表空间存储

![image](https://picturestore.nos-eastchina1.126.net/mysql/%E7%8B%AC%E7%AB%8B%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.jpg)

- DD表空间存储结构与一般的独立表空间存储结构是相同的
- 第4页(page 3)是SDI存储的root page, SDI以B+树进行组织
- 除了SDI信息外, 具体的元数据表也存储在这个表空间中, 存储方式与一般的表是一样的。

## 四. 总结与思考
1. 本文主要介绍了Mysql数据字典的概念以及数据字典的存储, 这里我们简单总结一下,
- 系统表以及我们创建的表, 元信息都存储在数据字典中
- 我们自己创建的表的元信息存储在系统表中, 同时也会序列化到自己的表空间中
- 系统表的元信息, 存储在系统表的数据字典对象中, 这些信息会被序列化到mysql.ibd文件中
- SDI信息以B+树的方式进行组织
2. 思考
- DD是数据库对象的合集, Mysql server层以及InnoDB层都需要这些信息, 他们是如何具体操作的?
- DDL的原子性是如何实现的?

## 五. 参考
1. https://dev.mysql.com/doc/refman/8.0/en/data-dictionary-schema.html
