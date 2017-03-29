---
layout: post
title:  "如何在数据量超大的表新增字段"
categories: Mysql
tags:  Work Mysql
author: Lzg
---

* content
{:toc}

这两天工作中有个需求需要给一个表增加一个字段，查了一下有怎么多数据：
`select count(1) from tableName` 大概有23165138条...
如果直接通过`alter`的话速度会很慢，所以pass掉．

方案－：
通过查询[stackoverflow](http://dba.stackexchange.com/questions/44777/how-to-add-column-to-big-table-in-mysql)得出：
  1. `SET FOREIGN_KEY_CHECKS = 0;  SET UNIQUE_CHECKS = 0;
  SET AUTOCOMMIT = 0;`
  2. 将原先的大表建表语句复制下来：`show create table tableName`, 然后修改表名`alter table nameA rename nameB`
  3. 建立新表，利用刚刚复制的语句．
  4. `insert into nameA(field1, field2....) select field1, field2 from naemB `;(别忘了手动ｃｏｍｍｉｔ！)
  5. `SET UNIQUE_CHECKS = 1;SET FOREIGN_KEY_CHECKS = 1；`


方案二：
　笨重一点的方法是直接将表的数据dump出去，然后加字段

方案三;
  写一个sql脚本，使用一的方式分段插入数据；




  欢迎各位看官交流分享：lzgsshen@163.com :smile:
