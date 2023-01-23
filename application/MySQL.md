# MySQL

[TOC]



## 基础篇



### 基础架构：一条SQL查询语句的执行过程

```mysql
select * from T where ID = 10;
```



基本架构示意图：

- Server层：所有跨存储引擎的功能
  - 连接器
  - 查询缓存（MySQL8删除）
  - 分析器
  - 优化器
  - 执行器
- 存储引擎层：插件式架构

![](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)





**连接器**

功能：

- 建立与客户端的连接
- 获取权限
- 维持和管理连接

```shell
mysql -h$ip -P$port -u$user -p
```



```mysql
# 显示连接 
show processlist;
```

连接超时：客户端超过wait_timeout的时间没有操作就会被断开 默认是8小时

连接类型：

- 长连接：

  客户端建立连接成功后 之后的查询一直用一个连接

  弊端：

  - 现象：导致内存不断增长，SQL执行过程临时使用的内存管理在连接对象中，内存资源在连接断开时才释放

  - 解决方法：
    - 定期断开长连接 释放资源
    - 执行`mysql_reset_connection` 不会重连和重新做鉴权 重新初始化连接资源

- 短连接：

  客户端建立连接成功后 执行几次查询后就断开 下次再重新连接

  弊端：建立连接过程复杂 耗费资源



**查询缓存** 8.0中删除

key-value

SQL语句 - 查询的结果



不建议使用查询缓存的原因：

当一张表的数据更新时，这个表对应的所有查询缓存都会被清空

所以当场景是更新比较频繁时，缓存命中率很低

静态表比较适合查询缓存

8.0之前支持按需使用:

- `query_cache_type = 2/DEMAND` 设置类型 `show variables like "query_cache_%"`
- `select SQL_CACHE * from T where ID = 10`  语句中指定



**分析器**

字符串 -> 用户要做什么

- 词法分析

  select -> 查询语句

  T -> 表名

  ID -> 列名

- 语法分析： `You have an error in your SQL syntax`

  判断SQL语句是否符合语法，关注`user near`后的内容



**优化器**

确定该怎么做

确定执行方案，如索引选择，表连接时的连接次序



**执行器**

1. 先判断用户对表有没有执行查询的权限
2. 有权限则打开表继续执行
3. 打开表时 根据表的引擎定义 调用引擎提供的接口
4. 调用InnoDB引擎取表的第一行 判断ID值是否为10 不是则跳过 是则将行的结果存储在结果集中（表中没有设置索引） （硬盘 -> 内存）
5. 调用引擎获取表的下一行(有索引的话 是满足条件的第一行和下一行)  直到最后一行
6. 将结果集返回给客户端





### 日志系统：一条SQL更新语句是如何执行的

更新语句涉及到两个重要的日志模块：

- redo log 重做日志
- binlog 归档日志



**redo log**

属于innodb引擎

赊账时的粉板和账本: redo log与数据文件

WAL: *write-ahead logging*  当有更新操作时 先写日志 再在空闲的时候写磁盘

crash-safe: 即使数据库异常重启 之前提交的记录也不会丢失 因为有redo log

![](https://static001.geekbang.org/resource/image/16/a7/16a7950217b3f0f4ed02db5db59562a7.png)

write pos -> check point之间的这段空间是可以写日志的空间

空间不够了就需要推进check point（将记录更新到数据文件）



**binlog**

归档日志，属于Server层



redo log与binlog的区别：

- redo log是innodb引擎特有的 binlog是server层的 所有引擎都可以使用
- redo log是物理日志 记录在某个数据页上做了什么修改 binlog是逻辑日志 记录语句的原始逻辑：给ID = 2的一行的c字段加上1
- redo log是循环写的 空间有限, binlog是追加写的 不会覆盖之前的日志



```mysql
update T set c = c + 1 where ID = 2;
```

更新语句在执行器和引擎阶段

1. 执行器调引擎接口 取ID = 2这一行 引擎根据树查找或者内存缓存中查找出 该行的数据 返回给执行器
2. 执行器根据返回的行数据 做更新操作 得到新数据 调引擎接口写入新数据
3. 引擎先更新数据到内存中 再记录到redo log 此时redo log状态为prepare状态
4. 执行器生成这个操作的binlog 并将binlog写入磁盘
5. 执行器调用引擎的提交事务接口 redo log 状态变为commit状态 更新完成

![](https://static001.geekbang.org/resource/image/2e/be/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)



**两阶段提交**

需要恢复数据 = 全量备份 + 全量备份的时间点以来的所有binlog



如果不采用两阶段提交会导致数据库现有数据和用binlog恢复的数据的不一致

- 先写redo log 再写binlog 写完redo log之后 crash了

  重启时redo log生效 但是binlog中没有记录 所以之后用binlog时没有这条操作

- 先写binlog 再写redo log 中间crash

  重启之后没有这个事务 但是之后用binlog恢复会多出一个事务



### 事务隔离

事务是在引擎层实现的，保证一组数据库操作要么全部成功 要么全部失败



**隔离级别**

- 读未提交：一个事务还没有提交时 它做的修改能被其他的事务看到
- 读已提交：一个事务提交后 它的修改才能被其他事务看到
- 可重复读：一个事务在执行过程中看到的数据总是跟这个事务**在启动时**看到的数据是一直的
- 串行化：对同一行记录，读写互斥，后访问的事务必须等前一个事务执行完成后才能继续执行



例子：

```mysql
create table T(c int) engine=InnoDB;
insert into T(c) values(1);
```



![](https://static001.geekbang.org/resource/image/7d/f8/7dea45932a6b722eb069d2264d0066f8.png)

|          | V1   | V2   | V3   | 实现                                |
| -------- | ---- | ---- | ---- | ----------------------------------- |
| 读未提交 | 2    | 2    | 2    | 没有视图概念 直接返回记录上的最新值 |
| 读已提交 | 1    | 2    | 2    | 每个SQL语句执行时创建视图           |
| 可重复读 | 1    | 1    | 2    | 事务开始时创建视图                  |
| 串行化   | 1    | 1    | 2    | 加锁                                |



**事务隔离的具体实现**

每条记录更新的时候都会同时记录一条回滚操作

![](https://static001.geekbang.org/resource/image/d9/ee/d9c313809e5ac148fc39feff532f0fee.png)

MVCC:同一个记录在系统中可以存在多个版本

当不存在比这回滚日志更早的read-view的时候，回滚日志就会被删除

read-viewA要获取它的值 就必须根据当前值依次执行回滚操作



**事务的启动方式**

长事务的弊端：

- 回滚日志一直得不到清理 过大
- 占用锁资源



建议显示启动事务：`begin + commit`



### 索引（上）

跟书的目录一样 索引的出现就是为了提高数据查询的效率



**索引模型**：用于提高读写速度的数据结构

- 有序数组：等值查询和范围查询都适用，但维护数据结构的成本太高
- 哈希表：只适用于等值查询 ，不适用与范围查询
- 查找树：等值查询和范围查询都适合，维护数据结构的成本相对不高（保持查找树的平衡）



**InnoDB的索引模型**

表时根据主键顺序以索引的形式存放的，一个索引对应一颗B+树

```mysql
create table T (
	id int primary key,
  k int not null,
  name varchar(16),
  index (k)
) engine=InnoDB;
insert into T(id, k) values(100,1),(200,2),(300,3),(500,5),(600,6);
```

两个索引：

- 主键索引：叶子结点存储整行数据，尽量使用主键进行查询 避免回表
- 非主键索引：叶子结点存储主键的值

![](https://static001.geekbang.org/resource/image/dc/8d/dcda101051f28502bd5c4402b292e38d.png)

Rn:为该行对应的所有数据



**索引维护**

新增数据时也需要维护索引的有序性，当页满时需要进行页分裂，当页利用率很低时会进行页合并

页分裂会：

- 占用性能
- 降低空间利用率 一个页的数据分到两个页 空间利用率降低了50%



规范建议：使用自增主键`NOT NULL PRIMARY KEY AUTO_INCREMENT`

使用自增主键的原因：

- 性能：

  插入记录时 系统会自动获取ID的最大值作为下一条记录的ID值，这样每次添加新纪录都是追加操作，如果是业务逻辑的字段作为主键，则不容易保证有序插入，数据写入成本较高

- 存储空间：

  非主键索引中的叶子结点都是主键 如果是业务字段所用字节数可能 > 整型所用字节数

  主键长度越小 -> 非主键索引叶子结点越小 -> 非主键索引所占空间越小





### 索引（下）

```mysql
select * from T where k between 3 and 5;
```

执行以上语句需要回表



**覆盖索引**

```mysql
select id from T where k between 3 and 5;
```

不需要回表，因为索引k的叶子结点上已经有我们想要的



如：建立(身份证号，姓名)的联合索引，可以在通过身份证号查询姓名的场景下避免回表



**最左前缀原则**

最足前缀：联合索引的最左N个字段，字符串索引的最左N个字符



**索引下推**

在索引遍历过程中，对索引中包含的字段先做判断，如果条件不满足就不对该行记录进行回表，从而减少回表次数

```mysql
select * from tuser where name like '张%' and age=10;
```

 

![](https://static001.geekbang.org/resource/image/76/1b/76e385f3df5a694cc4238c7b65acfe1b.jpg)



### 全局锁和表锁

**全局锁**

```mysql
# 给整个数据库实例加锁
flush tables with read lock
```

加锁后以下操作都会被阻塞：

- DML
- DDL
- 事务的提交



使用场景：做全库逻辑备份



需要一致性视图，但不想加锁：在可重复读隔离级别下开启一个事务

```shell
mysqldump -single-transaction
```

前提：引擎支持这个隔离级别，且所有表使用事务引擎



**表级锁**

一般在数据库引擎不支持行锁的时候会被用到

```mysql
lock tables ... read/write;
unlock tables; # 或者在客户端断开时自动释放
```



限制：

```mysql
# 线程A执行以下语句后 其他线程写t1 读写t2的语句都阻塞 在自己unlock tables前 也只能执行读t1 读写t2, 也不能访问其他表
lock tables t1 read, t2 write;
```



**MDL: metadata lock** ： 在访问表的时候自动加上

- DML时 对这张表加MDL读锁 MDL锁在语句执行时申请 但是在事务提交之后才释放
- DDL时 对这张表加MDL写锁



### 行锁

> 事务A更新了一行 事务B也要更新同一行则需要等待事务A操作完成后



**两阶段锁**

行锁在需要的时候加上，但是释放是在事务结束的时候

所以可以把最有可能造成锁冲突的热点行的修改放在里事务结束最近的地方



**死锁和死锁检测**



![](https://static001.geekbang.org/resource/image/4d/52/4d0eeec7b136371b79248a0aed005a52.jpg)

出现死锁之后的策略：

- 等待资源 直到超时 `innodb_lock_wait_timeout` 默认为50s
- 主动死锁检测：`innodb_deadlock_detect=on`检测是否有环，需要消耗大量的CPU资源



解决热点数据更新问题的方案：

- 如果能确保不发生死锁，可以关掉死锁检测
- 控制并发度：需要坐在数据库服务端
  - 中间件控制并发
  - 修改MySQL源码：同行修改 进入引擎前需要先排队
  - 一行改成逻辑上的多行（查询时汇总多行的数据）



### 事务

问题：

在可重复读下，事务开启时会创建一个视图，但是当这个事务中要修改一个行时 需要获取行锁 此时更新时获取到的数据是什么（一致性视图中的 还是当前数据）

```mysql
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `k` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
insert into t(id, k) values(1,1),(2,2);
```



![](https://static001.geekbang.org/resource/image/82/d6/823acf76e53c0bdba7beab45e72e90d6.png)



MySQL中的视图：

- `create view` ： 用查询语句定义的虚拟表，在调用的时候生成查询语句生成结果
- `MVCC的consistent read view`： 用于支持读提交和可重复读的实现



**快照**

- 可重复读的隔离级别下，事务启动的时候就拍了一个快照

- 事务开启时会向MySQL系统申请一个全局递增的transaction id，

- 每行数据都有多个版本，当事务修改数据时，会生成一个新的版本，版本号就是事务的ID row trx_id，之前版本的数据由undo log计算出来，并不是物理上存在的



![](https://static001.geekbang.org/resource/image/68/ed/68d08d277a6f7926a41cc5541d3dfced.png)



事务启动时，会以当前所有已创建但是未提交的事务的ID 创建一个数组，数组中的最小值为低水位，当前系统中已经创建过的事务ID的最大值 + 1为高水位



视图数组：事务创建的时候 系统中所有已创建但是未提交的事务ID（包括自己）

一致性视图：视图数组 + 高水位



![](https://static001.geekbang.org/resource/image/88/5e/882114aaf55861832b4270d44507695e.png)

注：

能看到的数据：

- 当前事务修改的（最新数据的版本号与自己的事务ID一致）
- 小于低水位的版本号的数据（已提交）
- 小于高水位且不在视图数组中的数组（已提交）



例如：

当前事务为9 事务有 5 6 7 8 其中 5 6 8 未提交， 7 已提交

则视图数组为[5,6,8,9] 高水位为10



**更新逻辑**

更新数据都是先**当前读**然后写，在最新的数据版本上进行修改

当前读时，如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待





## 实践篇



### 普通索引和唯一索引



**查询过程**

![](https://static001.geekbang.org/resource/image/1e/46/1ed9536031d6698570ea175a7b7f9a46.png)

如：

```mysql
select id from t where k = 5;
```

1. 从B+树 根结点开始 按层搜索到叶子节点
2. 走到右下角的数据页
3. 在数据页内用二分法定位记录
   - 普通索引，查找到(5,500)后 继续向后判断 直到找到第一个不满足条件的记录
   - 唯一索引，查找到(5,500)后 直接返回

对性能的影响：微乎其微

因为MySQL读取数据到内存的单位是页，向后比较操作是内存操作（绝大部分情况下 除非是该页的最后一个数据）



**更新过程**

`change buffer`： 只涉及普通索引的数据更新，数据的更改会先写到chang buffer中，不需要直接从磁盘读入这个数据页，之后查询需要访问到这个页的时候，再加载页到内存中，更新上change buffer中的修改（merge）



例子：插入一条(4,400)的新纪录

- 情况一：要更新的数据页就在内存中（区别不大）
  - 唯一索引：找到3-5之间的位置，判断到没有冲突才插入这个值
  - 普通索引：找到3-5之间的位置，直接插入
- 情况二：要更新的数据页不在内存中
  - 唯一索引：需要将数据页读入内存（涉及到随机读），判断没有冲突才插入
  - 普通索引：更新记录在change buffer中



**change buffer的使用场景**

写多读少，账单类、日志类数据

如果写了之后立即读，则会立即触发merge



**change buffer和redo log**

- change buffer：节省随机读磁盘的消耗
- redo log：节省随机写磁盘的消耗（转化为顺序写 写redo log）



### 为什么会选错索引



```mysql
# 创建表
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB;

# 插入10W行数据
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000)do
    insert into t values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

# 分析下列语句执行选择的索引
explain select * from t where a between 10000 and 20000;
```

![](https://static001.geekbang.org/resource/image/1e/1e/1e5ba1c2934d3b2c0d96b210a27e1a1e.png?wh=936*344)



此时会选择全表扫描而不是走索引a

因为扫描行数不够低，且这个扫描索引需要回表，所以优化器认为直接扫主键索引比较快



```mysql
# 记录所有执行时间超过long_query_time的查询
set long_query_time=0;
select * from t where a between 10000 and 20000; /*Q1*/
select * from t force index(a) where a between 10000 and 20000;/*Q2*/
# 慢日志存储位置: /var/lib/mysql/hostname-slow.log
```



**优化器选择索引逻辑**

因素：

- 扫描行数
- 是否需要使用临时表
- 是否需要排序





扫描行数受索引统计影响

索引统计是采样统计，抽几个页，计算数量，然后除以百分比（抽的页/总的页）

当变更的数据行数超过一定范围时，自动重新进行索引统计

```mysql
# Cardinality基数 表示一个索引中不同数的数量
show index from t;
# 重新统计索引信息
analyze table t
```





### 给字符串字段加索引

场景：表字段中有邮箱这类字符串字段

```mysql
mysql> create table SUser(
ID bigint unsigned primary key,
email varchar(64), 
... 
)engine=innodb; 
```



选择：

- 取全部字节

  占用空间大 但是扫描的行数少

- 取部分字节（前缀索引）

  占用空间少 但会增加扫描的行数（行数指的是访问的主键索引中的叶子结点数）

  而且前缀索引不能支持覆盖索引功能 会回表

  ```mysql
  select id,email from SUser where email='zhangssxyz@xxx.com';
  ```

- 当前缀区分度不够时 倒序存储 查询的时候也倒序（不适合范围查询）

- 对字符串进行hash 同样因为哈希冲突 所以需要在查询的时候也加上字符串的判断（适合等值查询 不适合范围查询）

```mysql
mysql> alter table SUser add index index1(email);
或
mysql> alter table SUser add index index2(email(6));
```



建立索引时，需要关注索引的区分度，统计索引上有多少个不同的值来判断使用多长的前缀

```mysql
# 计算该字段上不同的值的个数
mysql> select count(distinct email) as L from SUser;

# 计算出不同长度下的值 找出 >= L * 95%
mysql> select 
  count(distinct left(email,4)）as L4,
  count(distinct left(email,5)）as L5,
  count(distinct left(email,6)）as L6,
  count(distinct left(email,7)）as L7,
from SUser;
```



### MySQL有时候会“抖”一下

MySQL更新时候做的操作：

- 更新内存中的数据页（如果内存中没有该数据页就从磁盘中加载）
- 写redo log（顺序写盘）
- 返回给客户端



这样会造成内存中的数据页与磁盘中的数据页内容不一致，出现脏页，所以需要刷脏页(flush)



需要刷脏页的场景：

- redo log满了 系统会停止所有更新操作 将checkpoint往前推，将清除的redo log对应的脏页都flush到磁盘中

  ![](https://static001.geekbang.org/resource/image/a2/e5/a25bdbbfc2cfc5d5e20690547fe7f2e5.jpg)

- buffer pool内存不足 需要淘汰页 当淘汰掉的数据页是脏页的时候 需要刷盘
- 空闲的时候
- 服务正常关闭的时候



会影响性能的场景：

- 一个查询要淘汰的脏页太多
- redo log写满 更新暂停



所以需要控制脏页的比例

刷脏页的策略：

- 脏页比例
- 硬件的刷盘速度





### 删除掉表数据后 表文件大小不变

实质：数据删除了 但是表空间没有回收

InnoDB表的存储：

- 表结构：frm
- 数据: ibd



**数据删除流程**

删除只是把对应的记录或数据页标记为**可复用**

可复用而没有使用的空间 就像空洞一样



会造成空洞的情况：

- 删除记录
- 随机插入数据时数据页分裂（空间利用率降低）
- 修改索引上的值：意味着在非主键索引上删除旧值和插入新值



**重建表**

去掉空洞 提高空间利用率

```mysql
alter table t engine = InnoDB; # recreate操作
```

![](https://static001.geekbang.org/resource/image/02/cd/02e083adaec6e1191f54992f7bc13dcd.png)



问题：当往temp表插入数据时 如果新的数据写到了表A 就会造成数据丢失



Online DDL: mysql5.6开始

1. 先获取MDL写锁
2. 再退化为MDL读锁 允许其他增删改操作 但不允许其他修改表结构的操作

生成临时文件的时候 对A进行的操作记录 都放在了一个row log中

等生成完之后 再将row log应用在临时文件上

![](https://static001.geekbang.org/resource/image/2d/f0/2d1cfbbeb013b851a56390d38b5321f0.png)



inplace:创建的额外空间是在引擎中实现 对Server层而言感知不到



重建表的三种方式：

- analyze table 重新统计索引信息 没有修改数据
- alter table t engine = InnoDB 重新建表
- optimize table t = recreate + analyze





### count(*)的实现

InnoDB：

需要逐行将数据从引擎中读取 然后累积

逐行是因为需要比对版本 对当前事务可见的数据才累加

![](https://static001.geekbang.org/resource/image/5e/97/5e716ba1d464c8224c1c1f36135d0e97.png)





优化：

因为主键索引叶子结点是数据

普通索引叶子结点是主键值

遍历普通索引可以减少扫描的数据范围



如果要计数的话，则可以自己实现

方法：

- 缓存系统

  不可靠 因为无法保证原子性

- 数据库自身保存计数

  插入一行数据和在计数表中值+1放在一个事务中

  ![](https://static001.geekbang.org/resource/image/9e/e3/9e4170e2dfca3524eb5e92adb8647de3.png)



**不同的count用法**

count()是一个聚合函数 对于放进去的每个参数 判断如果不为NULL则 + 1

效率从高到低

1. count(*) 做了优化 不取值 直接按行累加
2. count(1)：server层对返回的每一行 放1 然后判断不为空
3. count(primary key) innodb遍历表 把每一行的id取出来 判断不为空 则+1
4. count(column) innodb遍历表 把每一行的对应字段取出来 判断不为空 则+1



### 日志索引相关问题



**两阶段提交的不同瞬间 MySQL异常重启 如何保证数据完整性**

![](https://static001.geekbang.org/resource/image/ee/2a/ee9af616e05e4b853eba27048351f62a.jpg)

时刻A：binlog没写 redo log没提交 所以事务会回滚

时刻B：写了binlog 但是redo log处于prepare 事务会提交

MySQL崩溃恢复的判断规则：

顺序扫描redo log

- redo log commit 直接提交事务
- redo log prepare 去检查对应的binlog是否存在并完整（通过一个XID字段关联） 
  - 如果存在且完整 则提交事务
  - 否则 回滚事务



binlog作用：

- 用于恢复库
- 用于高可用复制
- 用于其他下游系统消费



redo log作用：

- 顺序写盘 避免随机写盘
- 崩溃恢复



**数据最终落盘是从redo log更新过去的 还是buffer pool中更新过去的**

redo log中并没有记录数据页的完整信息 所以不存在从redo log直接更新到磁盘中的数据页的情况

- buffer pool中的数据页修改后 成了脏页 之后刷盘也是刷脏页
- 崩溃恢复的场景 判断某个buffer pool中的数据页存在对应的redo log 将其读到内存中 应用修改 成为脏页



**redo log buffer是什么**

```mysql
begin;
insert into t.. # 此时日志先先入内存中的redo log buffer
insert into t..
commit; # 提交的时候才写到redo log文件
```





### order by工作原理

场景：查询城市是“杭州”的所有人名字，并且按照姓名排序返回前 1000 个人的姓名、年龄。

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;

drop procedure icity5;
delimiter ;;
create procedure icity5()
begin
	declare j int;
  declare i int;
  set j = 1;
  while (j <= 10) do
  	set i=1;
  	while(i<=1000)do
    	insert into t(city,name,age) values('杭州', '大小7', i);
    set i=i+1;
  	end while;
  set j =j + 1;
  end while;
end;;
delimiter ;
call icity5();
```



```mysql
explain select city,name,age from t where city='杭州' order by name limit 1000;
```



`using filesort`表示需要排序

每个线程都有一个`sort_buffer`的内存空间 用于排序



**全字段排序**

1. 初始化sort_buffer 确认要放入city name age三个字段
2. 到索引city中依次找到满足where条件的主键ID
3. 根据主键ID到主键索引中找city name age的数据
4. 根据过滤条件 找到所有要排序的数据后 放入sort_buffer中进行快排（如果数据量大 也可能是借助磁盘进行外部归并排序）
5. 返回前1000个数据给客户端



```mysql
# 查看排序语句是否是外部排序 
/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */ 
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
# 查看次数的number_of_tmp_files是否不为0 不为0则表明使用了临时文件进行外部排序

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```



**rowid排序**

解决的问题：如果查询要返回的字段很多 sort_buffer中的字段过多 导致内存放不小 也会导致临时文件很多 排序性能不好

解决方法：当认为要返回的字段字节长度达到`max_length_for_sort_data`时 进行rowid排序 

1. 初始化sort_buffer 确定只放入两个字段 name id
2. 找出所有满足条件的name id 放入sort_buffer中
3. 排序 排序好后 遍历前1000行
4. 根据id到主键索引中取出city name age的字段 返回给客户端



**全字段排序和rowid排序**

rowid需要额外的回表操作（访问磁盘） 因此不会优先被选择 

思想：如果内存够用(sort_buffer) 就多用内存 尽量减少磁盘访问



共同点：原来的数据是无序的 所以需要排序



解决方式：提前将数据排序好 -> 联合索引

```mysql
alter table t add index city_name(city, name);
```

联合索引下的流程：

1. 从索引(city, name)找到第一个满足条件的主键id
2. 根据id到主键索引中去查找出city name age字段的值
3. 直到不满足条件或者到了1000条记录为止
4. 返回结果集



还存在的缺点：需要回表 

优化：覆盖索引

```mysql
alter table t add index city_name_age(city, name, age);
```

> 【Using filesort】 
>
> 本次查询语句中有order by，且排序依照的字段不在本次使用的索引中，不能自然有序。需要进行额外的排序工作。 
>
> 【Using index】 
>
> 使用了覆盖索引——即本次查询所需的所有信息字段都可以从利用的索引上取得。无需回表，额外去主索引上去数据。 The column information is retrieved from the table using only information in the index tree without having to do an additional seek to read the actual row. This strategy can be used when the query uses only columns that are part of a single index. 
>
> 【Using index condition】 
>
> 使用了索引下推技术ICP。（虽然本次查询所需的数据，不能从利用的索引上完全取得，还是需要回表去主索引获取。但在回表前，充分利用索引中的字段，根据where条件进行过滤。提前排除了不符合查询条件的列。这样就减少了回表的次数，提高了效率。） Tables are read by accessing index tuples and testing them first to determine whether to read full table rows. In this way, index information is used to defer (“push down”) reading full table rows unless it is necessary. See Section 8.2.1.5, “Index Condition Pushdown Optimization”. 
>
> 【Using where】 
>
> 表示本次查询要进行筛选过滤。



### 如何正确地显示随机记录

需求场景：从单词表的库中随机抽取三个单词

```mysql
mysql> CREATE TABLE `words` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `word` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=0;
  while i<10000 do
    insert into words(word) values(concat(char(97+(i div 1000)), char(97+(i % 1000 div 100)), char(97+(i % 100 div 10)), char(97+(i % 10))));
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```



**内存临时表**

```mysql
# 会用到临时表和排序
select word from words order by rand() limit 3;
```

采用的排序方式：

因为内存表的回表过程只需要访问内存 所以排序的字段越小越好 所以会选择rowid排序

语句的执行过程：

1. 创建一个临时表 引擎使用memory 两个字段 第一个字段R是double类型 第二个字段W是varchar(64)类型的 没有建索引

2. 遍历words表的主键索引 取出word值 对每个word值都随机生成一个0-1的double数 将这个二元组存入memory引擎中

   此时扫描行数为10000

3. 初始化sort_buffer 确定有两个字段

4. 从内存临时表中取出R值和位置信息(就是rowid) 这里涉及到扫描内存临时表 所以扫描行数变为20000 

5. 在sort_buffer中根据R值进行排序

6. 排序完成后 取出前三个数的位置信息 到内存临时表中取出word值返回给客户端 

   此时扫描行数+3 -> 20003

> 学习的方法:
>
> 先原理分析出扫描行数
>
> 再通过慢日志分析验证



**磁盘临时表**

当临时表的大小 > `tmp_table_size` 内存临时表 -> 磁盘临时表

```mysql
set tmp_table_size=1024;
set sort_buffer_size=32768;
set max_length_for_sort_data=16;
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* 执行语句 */
select word from words order by rand() limit 3;

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
```

当`order by limit k` 当k个元素的大小 < `sort_buffer_size`时会用到优先队列排序算法 （堆的空间用的就是排序的空间）

不需要临时磁盘文件 只需要取三个最小的值

1. 对于10000个准备排序的 (R, rowid)  取前三个元素构成一个最大堆
2. 去下一个元素 如果R小于堆顶元素 则替换
3. 直到10000个元素比较完成



**随机排序方法**

无需用临时表进行额外的排序

```mysql
# 总扫描行数为 C + Y1 + Y2 + Y3
mysql> select count(*) into @C from t;
set @Y1 = floor(@C * rand());
set @Y2 = floor(@C * rand());
set @Y3 = floor(@C * rand());
select * from t limit @Y1，1； //在应用代码里面取Y1、Y2、Y3值，拼出SQL后执行
select * from t limit @Y2，1；
select * from t limit @Y3，1；

# 优化：假设Y1，Y2，Y3是由小到大的三个数，则可以优化成这样，这样扫描行数为Y3 用索引避免避免后面两次的全表扫描
id1 = select * from t limit @Y1，1；
id2= select * from t where id > id1 limit @Y2-@Y1，1；
select * from t where id > id2 limit @Y3 - @Y2，1；
```





### SQL语句索引失效的场景



**在where语句中对索引字段进行函数操作**

场景：维护一个交易系统

```mysql
mysql> CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`), 
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

drop procedure idata; 
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=0;
  while i<3000 do
    insert into tradelog(tradeid, t_modified) values(i, concat('2019-', char(49 +(i % 9)), '-01'));
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```

需求：统计在所有年份中7月份发生的交易记录综述

```mysql
# 不会用到索引 因为会函数计算会破坏B+树中索引key的有序性 所以 优化器不会走索引根结点 会用全索引扫描
select count(*) from tradelog where month(t_modified)  = 7;
# 可以用到索引
select count(*) from tradelog where t_modified = '2018-7-1'

explain select count(*) from tradelog where (t_modified >= '2017-7-1' and t_modified <= '2017-8-1') or (t_modified >= '2018-7-1' and t_modified <= '2018-8-1') or (t_modified >= '2019-7-1' and t_modified <= '2019-8-1');

```

![](https://static001.geekbang.org/resource/image/3e/86/3e30d9a5e67f711f5af2e2599e800286.png)



**隐式类型转换**

字符串和数字比较时，字符串可以转化成数字

相当于往索引字段上调用了函数 

```mysql
mysql> select * from tradelog where tradeid=110717;
# 两个语句等同
mysql> select * from tradelog where  CAST(tradid AS signed int) = 110717;
```



**隐式字符编码转换**

```mysql
CREATE TABLE `trade_detail` (  `id` int(11) NOT NULL AUTO_INCREMENT,  `tradeid` varchar(32) DEFAULT NULL,  `trade_step` int(11) DEFAULT NULL, /*操作步骤*/  `step_info` varchar(32) DEFAULT NULL, /*步骤信息*/  PRIMARY KEY (`id`),  KEY `tradeid` (`tradeid`)) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into trade_detail values(1, 'aaaaaaaa', 1, 'add');insert into trade_detail values(2, 'aaaaaaaa', 2, 'update');insert into trade_detail values(3, 'aaaaaaaa', 3, 'commit');insert into trade_detail values(4, 'aaaaaaab', 1, 'add');insert into trade_detail values(5, 'aaaaaaab', 2, 'update');insert into trade_detail values(6, 'aaaaaaab', 3, 'update again');insert into trade_detail values(7, 'aaaaaaab', 4, 'commit');insert into trade_detail values(8, 'aaaaaaac', 1, 'add');insert into trade_detail values(9, 'aaaaaaac', 2, 'update');insert into trade_detail values(10, 'aaaaaaac', 3, 'update again');insert into trade_detail values(11, 'aaaaaaac', 4, 'commit');
```



查询id=2的交易的所有交易步骤

```mysql
mysql> select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2; /*语句Q1*/
```

预期的执行步骤：

1. 根据id在tradelog中对应的tradeid

2. 根据tradeid在trade_detail中找到所有条件匹配的行

   但是第二个步骤没有用到索引



因为两个表字符集不同 会发生字符的自动类型转化 所以导致了对索引字段进行了函数操作

解决方案：

- 修改两表的字符集 使其一致

- 修改SQL语句 强转condition中的值

  ```mysql
  mysql> select d.* from tradelog l , trade_detail d where d.tradeid=CONVERT(l.tradeid USING utf8) and l.id=2; 
  ```

  

```mysql
mysql> select * from trade_detail where tradeid=$L2.tradeid.value; 

# utf8 -> utf8mb4
select * from trade_detail  where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value; 
```



查询trade_detail里id=4的操作对应的操作者是谁

```mysql
mysql>select l.operator from tradelog l , trade_detail d where d.tradeid=l.tradeid and d.id=4;


select operator from tradelog  where traideid =$R4.tradeid.value; 

# 没有对索引字段进行函数操作了
select operator from tradelog  where traideid =CONVERT($R4.tradeid.value USING utf8mb4); 
```





### 只查一行也会执行慢的问题

```mysql

mysql> CREATE TABLE `t_slow` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000) do
    insert into t_slow values(i,i);
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000) do
    update t_slow set c = c + 1 where id = 1;
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```



**1.查询长时间不返回**



**等MDL锁**

```mysql
# 如果其他session持有MDL写锁 则该语句会阻塞住
select * from t where id = 1;

# 解决方法
show processlist;
# performance_schema=on 下
select blocking_pid from sys.schema_table_lock_waits;
```

![](https://static001.geekbang.org/resource/image/74/ca/742249a31b83f4858c51bfe106a5daca.png)





**等flush**

```mysql
# 关闭t 并将t内存的修改刷到磁盘上 但一般这个动作很快 所以是有其他的session正在打开表 所以该session无法关闭表
flush tables t with read lock;
```

![](https://static001.geekbang.org/resource/image/2b/9c/2bbc77cfdb118b0d9ef3fdd679d0a69c.png)



**等行锁**

```mysql
# lock in share mode 读锁 并且是当前读
# for update 写锁
mysql> select * from t where id=1 lock in share mode;

# 排查方法

mysql> select * from sys.innodb_lock_waits where locked_table='`test`.`t`'\G
# kill blocking_pid 断开这个连接
```

![](https://static001.geekbang.org/resource/image/3e/75/3e68326b967701c59770612183277475.png)



**2. 查询慢**

```mysql
# set long_query_time = 0
# 返回时间长
mysql> select * from t where c=50000 limit 1;

# 返回时间很长：一致性读 需要当前值 + undolo
mysql> select * from t where id=1；

# 返回时间很短：因为是当前读
select * from t where id = 1 lock in share mode;
```

![](https://static001.geekbang.org/resource/image/84/ff/84667a3449dc846e393142600ee7a2ff.png)





### 幻读

```mysql

CREATE TABLE `t_p` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t_p values(0,0,0),(5,5,5),(10,10,10),(15,15,15),(20,20,20),(25,25,25);


begin;
# 这条语句时怎么加锁的 在可重复读隔离级别下
# 全表扫描 给所有扫描到的行加上了行锁 还给行的两边空隙加了间隙锁
select * from t where d=5 for update;
commit;
```



**幻读**：

一个事务中两次查询同一个范围时 后一次查询看到了前一次查询没有看到的行（插入生成的）

可重复读隔离级别下 普通的查询时快照读 所以幻读在当前读下才会出现



**幻读的问题**：

- 语义：破坏的之前的语句的语义

- 数据一致性：数据库中的数据与用binlog还原的数据不一致

  把所有的记录都加上锁 也阻止不了新插入的记录

  ![](https://static001.geekbang.org/resource/image/dc/92/dcea7845ff0bdbee2622bf3c67d31d92.png)



```mysql
# binlog
update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/

insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/
```



**幻读解决方法**：引入间隙锁 （Gap Lock）

间隙锁之间不存在冲突 与间隙锁存在冲突的 是其他事务往这个间隙中插入一个记录的操作

![](https://static001.geekbang.org/resource/image/e7/61/e7f7ca0d3dab2f48c588d714ee3ac861.png)



间隙锁 + 行锁 = next-key lock ( , ]



间隙锁导致的死锁

![](https://static001.geekbang.org/resource/image/df/be/df37bf0bb9f85ea59f0540e24eb6bcbe.png)

1. sessionA id = 9不存在 所以加的是(5,10)的间隙锁
2. sessionB 也会加(5,10)的间隙锁
3. sessionB 被sessionA的间隙锁挡住了
4. sessionA 被sessionB的间隙锁挡住了



### 加锁规则

> 间隙锁在可重复读隔离级别下生效

规则：

- 加锁的基本单位是next-key lock (,] 间隙锁 + 行锁
- 在索引中访问到的对象才会加锁
- 索引上的等值查询，给唯一索引加锁时 next-key lock退化为行锁
- 索引上的等值查询 向右遍历且最后一个值不满足等值条件的时候 next-key lock退化为间隙锁
- 唯一索引上的范围查询会访问到不满足条件的第一个值为止



**等值查询间隙锁**

![](https://static001.geekbang.org/resource/image/58/6c/585dfa8d0dd71171a6fa16bed4ba816c.png)

id = 7

next-key lock -> (5, 10]

但是由于等值查询 向右遍历 id = 10 不满足条件 退化为间隙锁（锁的粒度变小了 并发度更高了）-> (5,10)



**非唯一索引等值锁**

![](https://static001.geekbang.org/resource/image/46/65/465990fe8f6b418ca3f9992bd1bb5465.png)

在索引c上给(0, 5]加next-key lock

因为不是唯一索引 所以会继续向右访问 10不符合条件 所以退化为间隙锁 (5, 10)

所以最终加锁范围是索引c的（0,10）又因为是覆盖索引且是读锁 所以不会锁主键索引（写锁会）



**主键索引范围锁**

![](https://static001.geekbang.org/resource/image/30/80/30b839bf941f109b04f1a36c302aea80.png)



等值查询访问到10 加next-key lock (5,10] 唯一索引 所以退化为行锁 10

向右遍历 范围查询 访问到15 加next-key lock所以最终范围为[10, 15]



**非唯一索引范围锁**

![](https://static001.geekbang.org/resource/image/73/7a/7381475e9e951628c9fc907f5a57697a.png)

等值查询 c = 10 加next-key lock (5,10] 因为是非唯一索引所以没有退化为行锁

再范围查询 访问到15 加next-key lock(10, 15]



**死锁的例子**

next-key lock：先加间隙锁 再加行锁

![](https://static001.geekbang.org/resource/image/7b/06/7b911a4c995706e8aa2dd96ff0f36506.png)



sessionA在索引C上 加了(5, 10] (10, 15)

sessionB在索引C上 先加了(5, 10)这个间隙锁 再加c = 10这个行锁 被sessionA持有的行锁 阻塞住了

sessionA再往(5,10)这个间隙加锁 被sessionB的间隙锁锁住

导致死锁





### MySQL如何保证数据不丢

只要保证binlog和redo log都持久化到了磁盘 就能保证crash safe



**binlog的写入机制**

1. 事务执行中：日志写到binlog cache
2. 事务提交时：binlog cache写到binlog文件（页缓存）中 同时清空binlog cache

![](https://static001.geekbang.org/resource/image/9e/3e/9ed86644d5f39efb0efec595abb92e3e.png)

- 每个线程都有自己的binlog cache 但是共用一个binlog file
- write只是把日志写到文件系统的page cache（还是内存）
- fsync 才是数据持久化到磁盘



**redo log的写入机制**

redo log的三种状态

![](https://static001.geekbang.org/resource/image/9d/d4/9d057f61d3962407f413deebc80526d4.png)

- 在redo log buffer 在内存

- write到磁盘 在page cache

- fsync到磁盘 在磁盘（两阶段提交的prepare阶段）

  持久化场景：

  - 事务提交
  - 后台线程每隔1秒的定时任务



双1配置：一个事务完整提交前 需要等待redo log 和binlog的刷盘

`sync_binlog = 1`

`innodb_flush_log_at_trx_commit = 1`



**组提交机制**

LNS *log sequence number*

单调递增的代表redo log的写入点

有三个**并发**事务，在trx1刷盘的时候 发现trx2 trx3的redo log也在

所以一起刷（批处理），所以延迟fsync的时间可以让更多的trx在一个组内

![](https://static001.geekbang.org/resource/image/93/cc/933fdc052c6339de2aa3bf3f65b188cc.png)





WAL：

- redo log和binlog 都是顺序写盘
- 组提交机制 可以大幅降低磁盘的IOPS消耗





### MySQL如何保证主备一致

**MySQL主备的基本原理**

*M-S结构*

![](https://static001.geekbang.org/resource/image/fd/10/fd75a2b37ae6ca709b7f16fe060c2c10.png)

主库：读写

备库：只读



日志同步流程：

备库B与主库A之间维持了一个长连接

1. B: change master 设置主库的信息以及binlog偏移
2. B: start slave：启动两个线程 io_thread sql_thread
3. A检验信息 根据偏移 从本地读取binlog 发送给B
4. 备库B拿到binlog后 写到本地为 relay log
5. sql_thread读取 relay log解析日志 并执行



**binlog的三种格式**

- `binlog_format=statement` binlog为SQL语句原文

  有可能导致主备不一致

  ```mysql
  
  mysql> CREATE TABLE `t` (
    `id` int(11) NOT NULL,
    `a` int(11) DEFAULT NULL,
    `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `a` (`a`),
    KEY `t_modified`(`t_modified`)
  ) ENGINE=InnoDB;
  
  insert into t values(1,1,'2018-11-13');
  insert into t values(2,2,'2018-11-12');
  insert into t values(3,3,'2018-11-11');
  insert into t values(4,4,'2018-11-10');
  insert into t values(5,5,'2018-11-09');
  
  mysql> delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1;
  # 选择索引a的时候删除id=4 选择索引t_modified的时候id=5 
  ```

  

- `binlog_format=row` : binlog中记录了真实的操作的主键id

  很占空间 如删除10W行的sql会翻译成删除10万条delete

  越来越主流 因为方便恢复数据

- `binlog_format=mixed`：mysql会对可能造成主备不一致的SQL采用row 否则采用statement



**循环复制问题**

双M A与B互为主备

A更新了一条语句 binlog发给B

B执行更新语句 生成binlog也发给A

-  A B server-id必须不同
- 备库在收到binlog并更新的时候 生成的binlog的server-id不变
- 每个库收到binlog时 判断server-id是否是自己的 是直接丢弃





### MySQL怎么保证高可用

**主备延迟**

> 同一个事务 在备库完成的时间与在主库完成的时间的差值 t3 - t1
>
> 大部分是t3-t2 主要是备库中消费中转日志 relay log的速度 比主库生成binlog的速度慢
>
> 主备延迟越小 主库障碍的时候 服务恢复的时间就越短 可用性越高



与数据同步的3个时间点：

1. 主库A执行完一个事务 并写入binlog t1
2. 备库B接收完这个binlog t2
3. 备库B执行完成这个binlog t3



**主备延迟的来源**

- 备库机器性能比主库机器差
- 备库压力大
- 大事务：某个事务的执行时间比较长
- 备库的并行复制能力



**主备切换的策略**

- 可靠性优先：一般是这个
- 可用性优先

可靠性优先：

双M结构下 主备切换的流程：

1. 判断B seconds_behind_master 是否<5 小于则进入下一步 否则持续重试

2. A -> readonly = true

3. 等待B SBM == 0

4. B -> readonly = false

5. 业务请求切到B

   (2-4之间 系统不可用)

![](https://static001.geekbang.org/resource/image/54/4a/54f4c7c31e6f0f807c2ab77f78c8844a.png)



可用性优先：

把步骤4 5调整到最开始执行 系统就不存在不可用时间 但是可能导致数据不一致



```mysql
mysql> CREATE TABLE `t` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `c` int(11) unsigned DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

insert into t(c) values(1),(2),(3);
```



```mysql
insert into t(c) values(4);
# 发生主备切换 假设此时主备延迟为5s
insert into t(c) values(5);
```



`binlog_format=mixed`时

由于主备延迟 备库B还没来得及执行`insert into t(c) values(4);`这个relay log就执行客户端的`insert into t(c) values(5);`

![](https://static001.geekbang.org/resource/image/37/3a/3786bd6ad37faa34aca25bf1a1d8af3a.png)



`binlog_format=row`时

会直接报主键重复错误

![](https://static001.geekbang.org/resource/image/b8/43/b8d2229b2b40dd087fd3b111d1bdda43.png)







****
