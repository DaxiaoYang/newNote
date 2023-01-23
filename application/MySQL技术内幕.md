# MySQL技术内幕

[TOC]

## MySQL体系结构和存储引擎

数据库与实例：

- 数据库：操作系统文件(磁盘中或者内存中)的集合
- 实例：数据库软件进程



MySQL组成：

- 连接池
- SQL接口
- 查询分析器
- 优化器
- 插件式存储引擎（基于表 也就是说每个表可以定义自己的引擎 插件式本身也是面向接口的思想）
- 物理文件



连接MySQL的方式:

> 本质上就是客户端进程与服务端进程的通信
>
> 进程通信方式：管道 命名管道 命名字 TCP/IP套接字 UNIX域套接字

- TCP/IP套接字：最常见

  连接到实例时 由mysql.user表负责权限校验

- 命名管道(windows下)和共享内存

- UNIX套接字：

  不是网络协议，用于客户端和服务端在同一台机器下的情况



## InnoDB存储引擎

![image-20220904195225801](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209041952894.png)

**内存池**：由内存块构成

- 存放线程需要访问的数据结构
- 作为磁盘读写的页缓存
- redo log buffer（本质也是对磁盘文件的内存缓存）



**后台线程**：（不是应对连接池中SQL请求的）

- 读：LRU保证内存中的页都是最近访问的
- 写：刷脏页
- 保证数据库出现异常后，InnoDB能正常恢复

**分类**：

- Master Thread：

  将缓冲池中的数据异步刷新的磁盘：刷脏页 合并插入

  UNDO页的回收

- IO Thread：

  负责处理异步IO的回调

- Purge Thread:

  回收undo页 因为事务提交后 其所使用的undolog可能不再需要

- Page Cleaner Thread:

  刷脏页



**内存组成**

- 缓冲池：对于磁盘随机读写的内存优化

  缓冲池实例可以配置多个 减少资源并发处理能力

  ![image-20220904202350477](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209042023507.png)

- LRU List Free List Flush List

  LRU list:

  单纯的LRU算法 在扫描索引下会产生很多非热点 但当时查询需要的页 从而将很多热点的页从LRU列表中移除

  解决方法：引入midpoint 新读取的页放入midpoint中 midpoint前的数据为活跃页(new list) midpoint后的数据为old list oldlist中的页需要经过一段时间后 才会被挪到new list中（有点像JVM 老年代）

  Free List:

  空闲页 数据库刚开始启动的时候 页都存放在这里

  Flush List:

  脏页列表 （脏页同时存在于Flush List和LRU List）

  > 那Flush list估计就是一个把脏页串起来的链表

- 重做日志缓冲：对于redo log文件的内存缓冲

  redo log buffer刷时机：

  - Master Thread：每秒
  - 每个事务提交的时候
  - redo log buffer剩余空间不足50%时

- 额外的内存池

  就是内存中的堆



**Checkpoint**

持久性的保证：

writer ahead log，事务提交的时候，先写redo log文件，再修改页，这样能保证宕机数据丢失的时候 可以通过redo log来恢复数据

CheckPoint解决的问题：

- 缩短数据库的恢复时间（减少redo log文件大小， checkpoint之前的页都已经刷到磁盘上 只需要恢复checkpoint之后的redo log）
- 缓冲池不够用(free list中没有空闲页) 将脏页刷新到磁盘
- redo log文件快满的时候 刷新脏页 让redo log文件中的空间可以被循环利用

Checkpoint做的事情：将缓冲池中的脏页刷到磁盘



Checkpoint分类：

- Sharp checkpoint：

  数据库正常关闭时，将所有的脏页都刷回磁盘

- Fuzzy checkpoint：每次只刷部分脏页到磁盘

  - master thread：

    每秒或每10秒从Flush list中刷新一定比例的脏页

  - flush_lru_list

    当LRU中空闲页不足时

  - async/sync

    当重做文件大小达到一定水位线时 从Flush list中异步/同步刷脏页

  - dirty page too much

    缓冲池中脏页的数量占到一定的比例的时候 刷新一部分脏页



**Master Thread工作方式**

```c
void master_thread()
{
  loop:
  for (int i = 0; i < 10; i++) 
  {
    do sth.
      sleep 1s
  }
  do sth.
  goto loop;
}
```

每秒一次的操作：

- 将redo log buffer中的内容刷新到redo log文件 即使事务没提交（这解释了即便很大的事务提交时间都是很短的 这是将一个大任务分摊在多个时间段内执行）
- 合并插入缓冲（IO次数少时执行）
- 刷脏页（脏页比例高时执行）
- 如果没有用户活动 切换到backgroud loop

每10秒的操作：（根据磁盘IO的能力 动态调整innodb_io_capacity来控制刷脏页和合并缓冲的数量）

- 刷脏页
- 合并插入缓冲
- 刷redo log buffer
- 删除无用的undo页



**InnoDB关键特性**

- **插入缓冲**: 对于非唯一的二级索引使用（唯一性索引 插入时需要去查询判断是否有重复的 这样每次插入还是会触发随机IO）

  表只有一个聚集索引的情况下 推荐主键按照递增的顺序插件 避免磁盘的随机读取（也有页缓存的因素 直接就在缓冲池中命中该数据页了）

  ![image-20220905201406618](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209052014732.png)

  当表还有**非唯一**的二次索引的时候，虽然主键插入时顺序的，但是二级索引的插入有可能不是（如果二级索引是时间的话 也是），还是会产生随机写盘

  解决方法：

  其实还是批处理，判断二级索引页是否在内存中，在则直接插入，不在则先放入一个Insert Buffer对象，之后再以一定的频率将Insert Buffer对象合入二级索引页中 这样就可以将多次随机IO 合并为一次随机IO

  实现：

  insert buffer是一个颗B+树，key：表空间id + 页偏移量

  将insert buffer中存储的二级索引的叶子结点的数据合并至索引页的时机：

  - 二级索引页被读取到缓冲池的的时候
  - 通过insert buffer bitmap 判断出该页已无可以用空间的时候
  - master thread合并

- 两次写（提升数据页的可靠性 insert buffer是提高性能）

  当写某个页写到一半数据库宕机时，需要先用页的副本来还原这个页，再用redo log来恢复

  ![image-20220905214107777](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209052141803.png)

  刷缓冲池中的脏页时，先将脏页copy到内存的doublewriter buffer 再写到磁盘double writer页（double writer页是连续的 所以是顺序写盘），最后再离散写入各个表空间文件中

- 自适应哈希索引：self-tuning

  建立条件：

  - 对某个页的查询条件(where后的语句)一直是一样的
  - 查询达到一定的次数
  - 只能应用于等值查询 不能用于范围查询

- 异步IO

  - 高效

    一个查询请求 需要进行多个页的IO操作 会将所有IO请求都全部发送出去 然后等待他们的完成 同步IO只能一个页的IO完成后 再进行下一个页的IO

  - 可以合并IO 将对某个页的连续空间的多次访问合并为一次IO

- 刷新邻接页

  当刷新某个脏页的时候，检查所在区extent（64页）的所有页，如果有脏页 一起刷新

  

## 文件

MySQL层面的：

- 参数文件
- 日志文件
- socket文件
- pid文件
- MySQL表结构文件

引擎层面：

存储引擎文件



**参数文件**：`my.cnf`

有默认启动参数，也可以通过参数文件进行配置

`show variables like '%aa%\G'`

参数类型：

- 动态

  可以在实例运行时通过 set命令更改 

  作用域分为global与session

- 静态

  运行时无法通过set修改，如数据存储目录



**日志文件**

- 错误日志

  `log_error`

- 慢查询日志

  执行时间超过`long_query_time`

  或者没有使用索引的`log_queries_not_using_indexes`

  的SQL语句都会被记录到慢SQL日志中`log_slow_queries`(默认值为10s)

- 查询日志

  所有的请求信息

- **二进制日志**

  记录了对MySQL数据库执行更改的所有操作

  作用：

  - 恢复：在全量备份文件恢复数据库后 使用binlog进行增量恢复
  - 复制：主备之间数据同步
  - 数据分析与审计

  与binlog相关的参数：

  - max_binlog_size：

    单个binlog文件的最大值 默认为1G

  - binlog_cache_size:

    基于会话 默认32K 开启一个事务 但未提交时 binlog写在内存中 当一个事务中的记录超过cache_size时 会使用临时文件记录（又一个避免大事务的原因）

  - sync_binlog:

    表示每写多少次内存缓冲写一次磁盘
    
  - binlog-ignore-db：
  
    表示不用同步的库
    
  - binlog_format:
  
    - statement：log里面是sql语句 某些条件下会导致主从不一致
    - row：表的行更改情况 row->隔离级别可以设置为read commited 并发性更好 但是row log占用空间更大
    - mixed：一般采用statement 特殊情况下采用row

- 表结构定义文件

  .frm



**InnoDB存储引擎文件**

- 表空间文件

  共享表空间：可以由多个文件组成

  独立表空间：每个表由 表名.ibd构成

- 重做日志文件

  循环写入

  ![image-20220918210400791](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209182104867.png)

  相关参数：

  - innodb_log_file_size:

    文件过大 导致恢复的时间会变长

    文件过小 会导致一个事务的日志需要多次切换重做日志(随机写盘 顺序写盘就是在同一个文件里面写)

redo log与binlog的区别：

- 所属范围：

  binlog是Mysql层面的 redo log是innodb引擎的

- 记录内容：

  binlog记录的是逻辑日志 sql/哪一行修改了什么内容

  redo log记录的是物理日志 涉及到具体的物理页

- 写盘的时间：

  binlog只在事务提交的时候写盘一次

  redolog会在事务进行中不断写redolog file 事务提交的时候也会写



## 表

**索引组织表**

InnoDB的数据都是按照主键顺序存放的

未显示指定主键时：

- 找一个表中的唯一非空索引
- 找不到则创建一个6B的键为主键



**InnoDB逻辑存储结构**

表空间 -> 段 -> 区  -> 页

![image-20220919203301798](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209192033839.png)



表空间：

- 共享表空间：

  undo信息 用于回滚

  插入缓冲索引页

  系统事务信息

  二次写缓冲

- 每个表独立的表空间

  索引

  数据

  插入缓冲bitmap



段：

- 索引段（非叶子结点）
- 数据段（叶子结点）
- 回滚段



区：

1MB 64个连续的页组成

刚开始建表的时候 为了节省空间 不会申请一个区的连续空间

未满32个页时 会申请32个碎片页 满了之后再申请64个连续页



页：

管理磁盘的最小单位 16k



行：

数据是按行存放的



**InnoDB行记录格式**

- compact 默认
- redundant 为了兼容以前的

Compact行记录格式

![image-20220919213740493](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209192137527.png)



- 变长字段长度列表

  用于表示每个变长字段的长度 每列最多2B 所以varchar类型的最大长度为65535

- NULL标识位

  指示该行数据中是否有NULL数据

- 记录头信息 5B

  next_record表明页中的行记录是链表形式构建的

  ![image-20220919214344524](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202209192143550.png)

- 事务ID列

- 回滚指针列

- 各个具体的列



Redundant行记录格式：

- 每个列的字段长度
- 记录头：next指针等
- 各个具体的列



行溢出数据：

不管是BLOB还是Varchar 没超过一定字节数 都不会放在外面 超过了都会放在外面

发生行溢出原因：一个页的大小为16K 16384字节 超出后B-tree Node页无法存放 所以数据只能存放在uncompress blob页中 原来的数据页中存放一些前缀数据和行溢出页的指针



char的行结构存储：

因为char(20)表示的20个字符 而不是二十个字节 所以当插入的信息不同时，具体占用的字节不同 所以行字段还是跟varchar一样要标识char类型的列有多少个字节



**InnoDB数据页结构**

![image-20221010084235527](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210100842599.png)



File Header: 页的头信息

- 页的checksum

- 表空间中页的偏移值

- 前后页的指针（数据页是双向链表 因为要支持从前往后和从后往前查两种操作）

- 页最后被修改的LSN

- 页类型

  ![image-20221010084832135](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210100848158.png)





Page Header: 页的状态信息

- 堆中第一个记录的指针，记录在页中是根据堆的形式存放的
- 指向可重用空间的首指针



Infimum和Supremum Record：页中主键的上下界



User Record: 实际存储行记录的内容



Free Space：记录为单位的空闲链表



Page Directory：页目录，记录主键的有序的稀疏索引

> B+树索引并不能找到某一条具体的记录 而是把记录所在的页加载到内存中 
>
> 在内存中先通过Page Directory定位到某个大概的范围 再通过链表指针寻找具体的记录



File Trailer：校验页的完整性



**约束**

关系型数据库跟文件系统的区别：可以保证存储数据的完整性

数据完整性：

- 实体完整性：主键
- 域完整性：保证每列的值满足特定的条件 如外键 唯一性约束 触发器 DEFAULT约束
- 参照完整性：外键 触发器



约束和索引的区别：

- 约束是用来保证数据完整性的逻辑的概念
- 索引时一个数据结构 有逻辑上的概念 也代表物理的存储方式



**视图**

没有实际的物理存储，只是一个逻辑的概念，查询视图等于执行定义视图的SQL

作用：

- 可以作为一层抽象 使用者不需要关心基表的结果
- 也可以起到封装的作用 保护基表 不单单是可以读数据 也可以设置为可更新视图 并且设置为只允许插入满足条件的数据



物化视图：

Oracle数据库支持，用于作预先计算（计算耗时较长的联表操作和聚合操作），空间换时间



**分区表**

MySQL数据库的表和索引可能由数十个物理分区组成

分区类型为水平分区（同一表中的不同行分到不同的物理文件中）



分区类型：

- RANGE：放某个连续区间的列值的记录放到不同的分区中 （最常用 如按时间分区）

  可以提高查询速度 过滤不需要查看的分区（跟clickhouse mergetree一样）

- LIST：离散的列值

- HASH分区：自定义哈希函数 使用哈希的目的是让数据均匀分配到预定义的各个分区中（有点像分库分表了）

  这个应该是单实例下的多文件存储

- KEY分区：MySQL提供的哈希函数



子分区：在分区的基础上再进行分区



分区中的null值：将null值视为最小的



分区与性能：（通常采用分库分表而不是分区表的原因）

分区适合的场景：OLAP，对某个时间段内的数据进行分析（partition pruning）只需要扫描对应的分区就行

分区带来的性能问题：

按某个列值进行分区（如主键）时，按主键的查询时可以加快查询时间，但是如果查询不按主键，查询时需要扫描所有的分区 IO数量随着分区的数量而增加



## 索引和算法

**B+树**

- 插入 （因为拆分页需要操作磁盘 所以会优先进行旋转 当前页满了 但是左右兄弟节点没有满）

  ![image-20221014073643499](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210140736572.png)

- 删除 （填充因子 一是为了磁盘利用率 二是为了磁盘IO效率 读一次页尽可能获得更多的数据）

  ![image-20221014074249677](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210140742714.png)



 **B+树索引的分裂**

页分裂不总是从中间记录开始，如果是按主键顺序插入的话，会浪费掉左边的叶子节点一般的磁盘空间

插入状态：

- 随机：取中间记录分裂开发
- 顺序插入：分配一个新的数据页给新插入的记录



**MySQL online DDL创建索引实现**

- 在创建索引时 将对原表的DML操作(insert update delete)的操作日志写到一个缓存上
- 待索引创建完成后，回放操作日志



**Cardinality值**

需要使用索引的情况：

- 查询的条件语句中用到
- 列数据的高选择性（字段的取值几乎没有重复 可以通过索引定位到很小范围的值）

高选择性判断：show index from t_table; 里面的Cardinality可以粗略地表示索引中有多少个不重复的值

Cardinality / rows 越接近1 建立索引效果越好 如果越接近0 建立索引效果越差

InnoDB中对Cardinality的统计：

- 统计条件：

  表中1/16的数据发送变化

  更新次数达到一定的值

- 统计方法：

  随机选取8个叶子节点的数据进行统计



**索引提示**：显示告诉优化器使用哪个索引

目的：

- 避免优化器错误选择索引
- 减少优化器选择执行计划的时间

语法：

- use index：只是提示优化器 非强制性
- force index：强制性



**Multi-Range Read**

目的：将磁盘的随机访问 -> 顺序访问

好处：

- 查询辅助索引 回表时 按照主键进行排序后 再到聚集索引中进行查找
- 减少内存中缓冲池中页被替换的次数（局部性原理）
- 将某些范围查询 拆分为键值对进行批量查询 从辅助索引中找出primary key时就过滤掉了不需要的部分



**Index Condition Pushdown**

从索引中取出数据时，就判断是否可以进行where条件的过滤，将where过滤下放到了引擎层（前提是索引数据中含有where中使用的字段）



**全文检索**

用倒排索引来实现，存储单词->单词所在文档的映射关系

- inverted file index：单词 -> 单词所在文档ID
- full inverted index：单词 -> 单词所在文档ID+文档内具体偏移



## 锁

锁分类：

- latch：在程序中用来保证并发安全的锁 分为互斥锁和读写锁 没有死锁检测的机制
- lock：以事务为单位进行获取和释放，在**事务**中锁定数据库中的对象表 页 行，仅在事务提交或者回滚后释放 有死锁机制



**InnoDB中的锁**

行级锁：

- 共享锁：允许事务**读**一行数据
- 排他锁：允许事务**删除或更新**一行数据



意向锁：划分锁的不同粒度

![image-20221024081811881](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210240818959.png)

- 意向共享锁：事务想要获取一张表中某几行的共享锁
- 意向排他锁：事务想要获取一张表中某几行的排他锁



**查询数据时的类型**

- 一致性非锁定读(默认的读取方式)

  用MVCC的方式读取数据 当读取的行加了X锁时，不会等待锁释放 而是会读取行的一个快照数据（当前数据 + undo log）commit之后才会产生一个新的快照版本

  不同的隔离级别下 对于快照的定义也不同：

  - read commited：读取最新一份快照数据

  - repeatable read：读取事务开始时的行数据版本

    

  ![image-20221025064124425](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210250641465.png)

- 一致性锁定读：

  - select xxx for update：对读取的行记录加X锁
  - select xxx lock in share mode：对读取的行记录加S

  

**自增长与锁**

自增长值的获取：

- auto-inc lock：自增长锁的获取和释放细化到以插入SQL语句执行完后释放 而不是事务提交后释放，这样在一张表中有大量插入语句需要执行就会串行化执行
- 通过互斥量对内存中的计数器进行累加的操作



**外键与锁**

场景：对子表中进行插入时，需要查询父表（一致性锁定读 加S锁）避免产生数据不一致

![image-20221025072625723](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210250726759.png)



**行锁的3种算法**

- record lock:锁单个行记录

- gap lock:锁一个范围 不包括锁记录

- next-key lock: gap lock + record lock 锁范围 包括锁记录（左开右闭区间） 设计的目的是为了解决幻读的问题

  在查询唯一性索引时 next-key lock会优化为record lock 提高并发度



**幻读问题**

定义：同一个事务中执行同样的SQL两次 会返回不同的结果 第二次会返回之前不存在的行



next-key locking作用：

- 解决幻读问题 锁住一个范围

- 实现唯一性检查：

  ```mysql
  select * from t where k=xxx lock in share mode;
  # if not found 此时已经这个事务已经锁定了k=xxx附近的范围
  # 多个事务并发插入只会有一个事务插入成功 其他事务会抛出死锁异常
  insert into t values(xxx)
  ```



**锁问题**

- 脏读：

  读到其他事务未提交的数据 read uncommited

- 不可重复读：mysql将其定义为幻读

  一个事务内同一SQL两次读到的数据是不一样的

  read commited

- 丢失更新：可以采用X锁避免应用层处理错误

  多线程update t set b = b + 1 的场景

  ```mysql
  # 对索引a xx前后加了next-key lock
  begin;
  select b from t for update
  where a = xx;
  update t set b = b + 1 where a = xx;
  end;
  ```



**阻塞**

- innodb_lock_wait_timeout：

  等待锁的时间 默认50s

- innodb_rollback_on_timeout:

  事务等待超时是否进行回滚 默认为不回滚（所以需要应用层进行手动处理）

  

**死锁**

概念：两个或两个以上的事务在争夺锁资源时互相等待的现象

解决方法：

- 将等待变为回滚 让事务重新开始 弊端是会影响数据库的并发性
- 超时 当一个事务的等待时间超时 进行回滚 弊端是可能导致undo log较多的事务被回滚
- 等待图 如果存在回路则证明有死锁 选择回滚undo量比较小的事务

出现概率相关因素：

- 事务数量
- 事务中操作的数量
- 数据的集合

![image-20221027072654303](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202210270726382.png)

会主动检测出死锁，innoDB会主动回滚事务





## 事务

引入事务的目的：

将将数据库从一种一致性状态转换为另一种一致性状态，事务提交时可以确保要么所有的修改都保存了，要么所有的修改都不保存



**事务**：

概念：

访问并更新数据中各种数据项的程序执行单元，其中的操作要么都生效，要么都不生效

特性：

- A 原子性：事务是不可分割的工作单位 事务中的所有操作执行成功 才算事务成功
- C 一致性：在事务开始之前和结束之后 数据库的完整性约束没有被破坏
- I 隔离性：事务提交前 其操作题其他事务不可见
- D 持久性：事务一旦提交 其结果是永久性的



分类:

- 扁平

  最常见 所有操作都属于同一个层次 

  不能回滚事务的某一部分的操作 出现错误只能回滚到开始 开销有点大

- 带有保存点的扁平

  允许事务在执行过程中回滚到同一事务中较早的状态 

- 链

  将多个事务连接起来 合并一个事务的提交与下一个事务的开始 下一个事务开始时将看到上一个事务的结果 效果就好像是在一个事务中执行的一样

- 嵌套（MySQL不支持）

  树形结构 事务的提交和回滚真正生效在父结点

- 分布式

  分布式环境下的扁平事务 需要访问网络中的不同节点



**事务的实现**

锁：隔离性

redo log：持久性 

undo log：原子性 一致性 帮助事务回滚 MVCC

redo log与undo log：

- 相通之处：都可以用作恢复操作 

  redo log可以在宕机时 用来恢复事务提交的页修改操作

  undo log可以将行记录恢复到之前的某个版本

- 不同之处

  redo log是物理日志 记录的是物理页的修改操作

  undo log是逻辑日志 根据每行记录的修改



**redo log** ：用于实现事务的持久性

组成：

- 内存中的redo log buffer
- 硬盘中的redo log file



**redo log与binlog**

- 产生的层级不同：

  redo log在innodb引擎层产生 binlog在mysql层产生

- 记录内容不同：

  binlog为逻辑日志 记录对应的SQL语句

  redo log为物理日志 记录对物理页的修改

- 写入磁盘的时间点不同

  binlog只在事务提交的时候写入

  redo log在事务进行中不断写入

- 幂等性

  binlog的修改操作不是幂等的 重复执行插入会产生多条日志

  redolog的修改操作是幂等的



**log block**：redo log buffer与redo log file的组成单元

大小与磁盘扇区一致 为512字节

格式：12(头) + 492  + 8（尾）

头：

- log block在数组中的下标
- 占有大小
- 块中第一个日志的偏移量
- 最后被写入时checkpoint的值

尾：

- 最后被写入时checkpoint的值



**log group**

是一个逻辑上的概念 由多个redo log file组成



redo log buffer -> redo log file

- 事务提交时
- log buffer中内存空间不足一半时
- log checkpoint时

> 并非完全顺序写入 redo log file
>
> 除了写log block外 还需要更新每个log group第一个log file前面2KB的信息



redo log格式：

- 类型
- 表空间ID
- 页偏移量
- body



**LSN**：8B 单调递增 对数据库的修改进度

- redo log写入的字节的总量

- checkpoint位置（刷新到磁盘的LSN）
- 页的版本（用来判断页是否需要恢复操作 页的LSN < 最后提交的事务的LSN 则需要应用redo log）



重启/宕机恢复的部分：

redo log LSN - checkpoint（磁盘写入的LSN）



**undo**

- 用于回退事务的修改：

  是逻辑(SQL层面)日志 insert -> delete delete -> insert

- MVCC：

  实现非锁定一致性读 最新记录 + 消费undo log

- 与redo log

  undo log跟数据库操作一样 也需要持久化 所以也会产生redo log



事务提交时：

- undo log放入列表 用于后续的purge
- 判断undo log所在的页是否可以重用



undo log格式：

- insert 

  只对事务本身可见 对其他事务不可见

  在事务提交的时候 可以直接删除

- update

  在事务提交时 放到undo log链表 等待purge线程删除



**purge** :清理

用于实际执行update和delete操作 删除记录

不在事务提交的时候直接删除的原因是因为要支持MVCC 在事务提交的时候 可能有其他事务正在引用该记录



![image-20221104052632326](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211040526418.png)

purge过程：

先从history list中找到最老的undo log

再定位到其所处的undo log page进行删除

因为相近提交的undo log可能在一个undo log page上 这样可以减少随机读取操作



**group commit**

目的：因为每次事务提交 都需要fsync 将redo log写入文件中 批处理的思想 想一次fsync刷新多个redo log到文件中



事务的提交的步骤：

1. 修改内存中事务对应的信息 同时写入内存中的redo log buffer
2. 调用fsync 将redo log buffer写入redo log file



**事务的隔离级别**

隔离级别越低 事务中持有的锁和持有锁的时间就越短

SQL标准：

- read uncommited

- read commited：binlog主从同步时 必须要使用row-format的 否则使用statement格式的binlog会发生数据不一致的问题

  ![image-20221105055309054](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211050553103.png)

  示例中的问题：

  - 没有gap lock锁住一个范围 导致master上的事务B可以插入成功
  - 使用statement的binlog 在slave先执行insert 再执行delete

- repeatable read：标准中的这个级别是不能解决幻读问题的 mysql通过next-key lock实现了在这个隔离级别下避免了幻读的问题

- serializable



**分布式事务**

> 允许多个独立的事务资源（关系型数据库）参与一个全局的事务中



**XA事务组成**：

- 多个资源管理器（SQL数据库）
- 一个事务管理器：与所有资源管理器通信
- 应用程序：定义事务的边界 指定全局事务中的操作

![image-20221106095718701](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211060957765.png)

两阶段提交：（编程语言中完成分布式事务的操作）

1. 资源管理器告诉事务管理器自己准备好提交了 prepare
2. 事务管理器告知所有的资源管理器 commit / rollback（只要有一个节点不能提交 都是rollback）



**内部XA事务**

常见与MySQL binlog与InnoDB redolog之间

事务提交时 先写binlog 再写redo log 两者需要同时成功或失败 

解决方式：两阶段提交

1. 事务提交的时候 innodb redo log先prepare(这样 如果2-3之间 数据库宕机了 重启的时候 判断状态 可以再进行redo log write)
2. binlog write
3. redo log write

![image-20221106101205800](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211061012846.png)



**不好的事务习惯**

- 在循环中提交事务（默认就是执行一次SQL commit一次的）

  会极大的影响性能 因为每次commit 都需要写binlog和redo log 所以最佳做法是在循环前开启事务 循环后结束事务

  担心循环太多造成长事务 可以尝试着把循环的次数切分开 划分为多个小事务

- 使用自动提交 

  最好是显示划分出事务的边界

- 在存储过程中使用自动回滚

  效果没问题 是可以回滚 但是报错的信息没有有效上报给应用程序 所以应该在应用程序中捕获SQL异常进行回滚 日志记录 告警等相关处理



**长事务**

场景：银行每隔一段时间 需要对所有账户计算利息

长事务的问题：

- 当出现异常时 回滚事务的代价太大（消费undo log）

- 占用过多的锁资源 并发能力下降

  

解决方法：转化为多个小批量的事务 这样回滚的单位较小 同时也可以看到具体的执行进度





## 备份与恢复

会导致数据丢失的情况：服务器宕机（不是数据库宕机） 磁盘损坏 RAID卡损坏

解决办法：进行数据备份



**备份类型**：

按备份与数据库运行的关系

- 热备：不影响正常的数据操作 在线备份

  1. 记录备份开始时 redo log LSN
  2. 复制共享表空间和独立表空间文件
  3. 记录备份结束时 redo log LSN
  4. 复制备份期间产生的redo log

- 冷备：停机备份 复制相关数据库物理文件即可

  文件：

  - frm 表结构文件
  - 共享表空间文件
  - ibd 独立表空间文件
  - redo log file
  - my.cnf 配置文件

- 温备：也是在数据库运行中进行 但是会有一定的影响 如会加一个全局的读锁

按备份后的文件内容

- 逻辑备份：SQL语句构成

  - 可读性强
  - 但是恢复时间长

  实现：

  - mysqldump
  - select ... into outfile 导出一张表的数据

- 裸文件备份：

  - 没有可读性
  - 但是恢复时间短

按备份的数据库内容：

- 完全备份

- 增量备份

  - 可以通过binlog 但是效率较底 

  - 减少备份的量的方法：

    当当前页的LSN<=全备时的LSN（表示全备中的页与当前页数据一致） 则该页不需要备份

- 日志备份：备份binlog

  



备份的一致性：MVCC + 备份时开启事务即可



**==快照备份==** ： 这个思路很好 快照就是那个时间点的数据内容

LVM的快照备份实现：写时复制

原理：

建立快照时 只是赋值了数据的元数据（数据结构等信息）

写：当原数据有修改时 在修改前将原始数值复制到快照区域

读取快照时：先读快照区域 读不到（表明没有修改）再读原始区域

![image-20221106115911999](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211061159053.png)



**复制**

原理：

1. master写binlog
2. slave将master的binlog复制一份为relay log
3. slave重做relay log达到与master的数据一致性

![image-20221108065651532](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080656649.png)



作用：

- 读取的负载均衡 将读取的请求派发到slave上 减轻master的负担
- 数据库备份 
- 高可用性和故障转移



## 性能调优

**CPU**

OLTP特点：

- 事务并发量大
- 事务处理的时间较短
- 查询语句比较简单 一般都走索引 复杂查询较少

所以对CPU要求不是很高，属于IO密集型的操作，多核对于增大并发量请求有帮助



**内存**

给数据库分配的内存最好可以覆盖热点数据的大小，可以通过缓冲池命中率来判断



**硬盘**

- 机械硬盘

  性能指标：寻道时间 + 转速

  因为顺序访问的速度远高于随机访问 所以数据库都在尽量利用这个特性

  优化：

  - 多块机械硬盘 组成RAID
  - 将数据文件分在不同的硬盘中 达到负载均衡的效果

- 固态硬盘

  读取速度跟内存是一个数据量级的

  ![image-20221108072040093](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080720143.png)

  

  

**RAID**

将多个硬盘组合起来 形成一个磁盘数组 提高性能 操作系统也会把RAID看成一个硬盘

作用：

- 增强容错能力 可以备份数据
- 增加处理率或容量



组合方式：都是trade-off

- RAID 0：

  多磁盘并列 并行IO 速度最快 无冗余数据 所以一个磁盘损坏 所有的数据都会丢失

  ![image-20221108073755113](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080737165.png)

- RAID 1

  由2个以上的硬盘互相做为镜像 数据安全性最高 但是写入速度慢（因为要写两份） 利用率最低 因为只能算一个磁盘的容量

  ![image-20221108074351166](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080743200.png)

- RAID 5：权衡了数据安全性 存储成本（空间利用率） 读取速度

  不直接备份数据 而是备份奇偶校验信息 当一个磁盘的数据损坏的时候 可以用校验信息 + 剩下的数据去恢复被损坏的数据

  ![image-20221108074930513](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080749549.png)

- RAID 10与RAID 01

  - 10：结合了RAID 0速度与RAID 1的安全性（数据库的最佳选择）

    最外层是RAID 0（分区） 底层是RAID 1（镜像）

    ![image-20221108075718161](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080757193.png)

  - 01：

    最外层是RAID 1 镜像 底层是RAID 0 分区

    ![image-20221108075900250](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080759299.png)

- RAID 50

  RAID 0 数据分区 + RAID 5校验信息

  ![image-20221108080336474](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202211080803507.png)



**RAID Write Back功能**

将写入的数据放入主机的缓存中 后续批处理 断电时会有电池备份







