+++
date = '2025-03-07T17:27:19+08:00'
draft = false
tags = ['MySQL', 'PostgreSQL']
title = 'MySQL 对比 PostgreSQL 使用上的的区别'
+++

以前工作中用的都是 MySQL。最近做银行的项目要求使用 PostgreSQL，这里对比一下常用的功能里，两种数据库的区别。
比较深入的实现原理部分这里就先不写了，后面会单独有文章来说。

这里的 Mysql 使用版本8.0, Pgsql 使用版本 14.7。

### 数据类型
比较简单的，比如类型名不同就不说了，Mysql常用的类型Pgsql都有相对应的类型。

Pgsql 还增加了 JSONB（二进制JSON）、数组（ARRAY）、范围（RANGE）、几何类型、网络地址类型等类型的支持。

#### 对 JSON 的支持
Mysql 的 JSON类型是以文本形式报错的，使用 JSON 格式的字段进行查询时需要将整个 JSON 文档解析。

Pgsql 中有 JSON 和 JSONB 类型，其中 JSONB 是以二级制格式保存，存储的时候消除了空格、重复键、顺序，对 JSONB 类型的字段进行查询时不需要解析整个文档，效率更高。

Pgsql 的JSONB类型支持 GIN 索引（倒排索引）对键和值进行索引。

Mysql 虽然支持通过虚拟列对 JSON 字段进行索引，但是只能针对指定字段索引，而且性能也不如 Pgsql 的GIN索引。

### 自增主键/序列
Pgsql 中主键自增基于序列（Sequence）实现，而 Mysql 中主键自增基于自增列（Auto Increment）实现。

Mysql自增列的使用比较简单，只需要在创建表时指定主键为自增列即可。
但是功能较为单一，并且在默认配置（innodb_autoinc_lock_mode=1）下，极端情况下并发插入时可能导致锁竞争而成为性能瓶颈。


而 Pgsql 中的序列使用无锁实现并发情况下的ID获取，性能更高。而且序列支持多个表共享唯一ID。

Mysql 在使用自增主键时，插入一个比当前最大主键值大的数时，会同时更新最大主键值，而 Pgsql不会。


这是一个很容易犯错的地方，切换到 Pgsql时如果不留神，可能会默认认为和 Mysql 的机制是相同的。修改后导致后续使用自增主键插入数据时报错。

### 默认隔离级别
Mysql 默认隔离级别为可重复读（REPEATABLE READ），而 Pgsql 中是读已提交（READ Committed）。

### 索引类型
Mysql 的众多索引类型中，使用的最多的也就是 B-Tree 索引了。

Pgsql 中除了也有 B-Tree 索引，还有 GIN 索引（倒排索引），支持对 JSONB、ARRAY、全文搜索向量（TSVECTOR）等类型创建索引。
特别是对JSONB类型的支持很有用。

### Pgsql不支持字段位置调整
Pgsql 新加的字段的顺序只能在最后面，不能调整。
这个不影响实际使用的功能，但是对于我这种有强迫症，必须把 created_at 和 updated_at 字段放在最后面的人来说，非常难受。

### 扩展性
Pgsql 支持扩展，比如地理信息、时序数据、向量类型以及分词等。非常丰富。

Mysql 的扩展非常少。