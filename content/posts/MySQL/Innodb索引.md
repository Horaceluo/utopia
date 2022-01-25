---
title: "Innodb索引"
date: 2022-01-25T14:26:05+08:00
draft: false
---

# 前言
MySQL Innodb引擎，无论是聚簇索引，还是二级索引，都是使用B+树来存储数据。B+树有以下特点：

- 非叶子节点不存储数据
- 叶子节点使用链表连接起来

## 聚簇索引和二级索引

从物理存储层面上来看，Innodb的索引可分为聚簇索引和二级索引。聚簇索引的非叶子节点存储的是数据主键信息，叶子节点存储了数据的信息。二级索引的非叶子节点存储的是索引信息，叶子节点存储了数据的主键信息。

从二级索引的与聚簇索引的却别也可以看出，二级索引之所以叫二级索引的原因是：二级索引不存储全部数据信息，而是存储数据的主键，这样在使用二级索引进行查询的时候，可能还需要根据查询到主键去查询数据--也就是通常说的“回表”操作。

## 复合索引

MySQL一个索引可以包含多个列(最多16列)。索引使用最左匹配原则。假如有以下数据表：

```sql
CREATE TABLE test (
    id         INT NOT NULL,
    col1  CHAR(30) NOT NULL,
    col2 CHAR(30) NOT NULL,
    PRIMARY KEY (id),
    INDEX col_index (col1,col2)
);
```

col_index索引使用了col1和col2两个列。对于包含了col1，col2的查询，都可以使用索引col_index来优化查询。

```sql
SELECT * FROM test where col1='A' AND col2='B';
SELECT * FROM test where col1='A';
```

上面都可以使用上索引。但是对于下面这个语句，由于不符合最左匹配原则，所以是不会用上索引的。

```sql
# 不会用上索引的语句
SELECT * FROM test where col2='A';
```

### 覆盖索引

还是上面的test表，如果增加一个col3字段，如果我们使用以下语句查询
```sql
SELECT * FROM test where col1='A';
```
MySQL会使用col1_index字段进行查询。由于是二级索引，所以在查询出col1='A'的数据的主键信息之后。还需根据主键在test表获取到col3的数据。

但是如果把查询语句改为下面这样子：
```sql
SELECT col1,col2 FROM test where col1='A';
```
由于col1,col2是复合索引col_index的数据。所以，在匹配到二级索引的数据之后，实际上就可以返回查询结果了。无需再根据主键去查询数据。可以省掉回表的消耗。

那么在建索引的时候，就需要考虑查询的场景，使得复合索引尽量覆盖查询条件、查询内容。使得索引得效果更佳明显。