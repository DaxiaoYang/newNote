# ClickHouse

[TOC]




## 发展史



**需求**：对Web流量进行统计分析 从固定维度的分析（生成固定报表）到需要不同维度的实时分析（生成动态报表）



**架构演变史**：

1. Mysql Mysima引擎 因为需要查询快且对事务要求不高 ROLAP
2. 自研了Merged 对所有维度进行预处理聚合 速度很快但是存在维度组合爆炸 以及只能生成固定报表的问题 MOLAP
3. 自研了server 数据库 采用了之前的merge与server结合的方式 HOLAP
4. 在server的基础上完善了功能 成为一个DBMS 即clickhouse ROLAP





## 架构设计



**核心特性**

- DBMS该有的都有：DDL DML 权限访问控制 数据备份与恢复（导入导出） 分布式管理 

- 列式存储 + 压缩：减少磁盘存储和IO压力

  压缩原理：遇到重复性的字节会用数字来表示 

  而列式存储把相同类型的数据放到一块 重复性的数据会增加 可以达到更高的压缩比

- 向量化执行引擎

  SIMD 单指令并行操作多个不同的数据

- 支持分区（多线程）和分片（分布式）

- 基于关系模型 支持SQL

- 存储引擎抽象化 接口与实现分离 有多种不同的存储引擎（借鉴MySQL）

- 多主架构

- 在线查询

  实时数据的聚合 不需要对数据进行预处理

- 数据分片与分布式查询

  表分为本地表（物理）和分布式表（只在逻辑存在）



**架构设计**：

column对象：表示一个列，封装了对数据的运算，提供对数据的读取能力

field对象：表示一个列中的一行数据

DataType: 数据的序列化与反序列化

block对象：内部的数据操作都是面向block对象 组合了column datatype 列名字符串

IStorage接口：指代table，根据AST对象，返回具体的数据，交给interpreter

parser：创建AST对象

interpreter：解释AST 执行查询 返回block对象

shard分片：一个节点有一个分片 



**设计思想**

- 自底向上：从具体的硬件出发 
- 优化每一个细节，采用根据不同的场景使用最佳的算法
- 迭代



## 安装与部署

```shell
首先，您需要添加官方存储库：

yum install yum-utils
rpm --import https://repo.clickhouse.tech/CLICKHOUSE-KEY.GPG
yum-config-manager --add-repo https://repo.clickhouse.tech/rpm/stable/x86_64
如果您想使用最新版本，请将stable替换为testing（建议您在测试环境中使用）。

然后运行这些命令以实际安装包：

yum install clickhouse-server clickhouse-client
```



**配置**

`config.xml`：全局配置

`user.xml`：用户配置



**客户端访问方式**

- 基于TCP的，性能更好，CLI方式（command line interface），其中的非交互式执行可以用于批处理(导入或者导出)

  `cat file.csv | clickhouse-client --query "INSERT INTO some_table FORMAT CSV"`

- 基于HTTP的，如JDBC，兼容性更好



**基准测试工具**

`clickhouse-benmark`

`echo "SELECT * FROM some_table LIMIT 100 | clickhouse-benchmark -i 5"`





## 数据定义

**数据类型**：

- 基本类型

  数值

  字符串：String FixedString UUID

  时间类型

- 复合类型：数组 元组 枚举 嵌套

- 特殊类型



**分区**：`partition by` 只有`MergeTree`系列的表支持数据分区

每个分区对应一个目录，分区可以用来加快查询，在修改和删除行数据的时候作为一个整体被删除和重新插入，分区粒度过细，导致分区过多会导致性能变差



**分区相关操作**

- 查询 分区信息存储在`system.parts`里面
- 删除
- 复制
- 重置
- 卸载和装载
- 还原与备份



**数据的写入方式**

- values
- format 指定格式
- select：将select查询结果写入到表中



**数据的删除与修改**

删除逻辑：

- 对每个分区目录都建立一个新的目录 重新往里面写数据
- 旧目录在下一次Merge中被删除





## 数据字典

key-value

会通过定时检查变化的机制，刷到内存中，方便查询





## MergeTree原理

**创建**

- partition by：分区
- order by：索引
- index_granularity：索引粒度



**存储结构**

分区目录下：

- 数据文件
- 元数据文件：
  - 校验和
  - 列信息
  - 一级索引（稀疏索引）
  - 标记文件
  - 分区相关
  - 二级索引相关



**数据分区**

分区可以用来减少数据的扫描范围（维护了分区中索引值的数据范围）

- 分区命名规则

  分区名\_最小数据块号\_最大数据块号\_合并次数

- 分区目录的合并

  每一批数据的插入都会生成一个分区目录 即便是同一个分区的数据

  写入后的10-15分钟 后台任务会将相同分区的目录合并 合并后生成新的分区目录

  旧的分区目录active=0 



**一级索引**

一级索引可以用来减少数据的扫描范围

稀疏索引 间隔由索引粒度决定

索引的查询过程：归并



**二级索引**

减少数据的扫描范围



**数据存储**

分区目录中的各列单独文件存储

数据文件由一个个压缩数据块构成：将读取粒度降低到压缩数据块级别

写入的批次与压缩数据快的关系不定



**数据标记**

一级索引->数据

1. 指定要找的压缩块 读到内存中解压
2. 根据偏移量读具体的数据



