[TOC]

## 认识Pulsar

**Pulsar的特点**：（相较于kafka rabbitMQ）

- 对云原生环境的适配

  消息队列属于有状态服务 运维成本高

  pulsar：broker无状态 bookie只存储数据 元信息放在zookeeper中

  可以很好地支持水平扩缩容

- 支持多租户和海量topic

- 可靠性与性能的平衡：性能瓶颈主要在磁盘和网卡

- 低延迟

- 高可用：自带各种容灾方案

- 轻量函数式计算：可以通过上传代码自定义topic之间的处理过程

- 可以同时支持流处理和批处理



**pulsar基本结构**

- 代理层：对下游的Broker做反向代理

- Broker层：

  无状态服务 负责业务逻辑

  处理请求

  - 管理流：

    对租户 命名空间 topic的管理 通过http

  - 数据流

    处理消息 通过protoBuf(最快的二进制序列化协议)

- Bookie层：

  负责存储数据 可水平扩展 容错 低延迟 只可追加数据的存储服务

- Zookeeper：

  存储Broker和Bookie的元数据和帮他们选主



## 客户端

**名词**：

- 租户：

  一个集群下有多个租户 

  一个租户下可以有多个命名空间 

- 分区topic：

  每个分区会落在一个Broker上

- 持久/非持久topic：

  非持久化的topic都只存在内存中



**Topic**

- 格式：

  {persistent | non-persistent}://tenant/namespace/topic
  每个topic/分区topic的每个分区都会归属到一个broker上

- 分区topic

  topic的分区 分布在不同broker上 以轮询方式生产/消费 分摊流量压力 对用户无感知

- 创建：

  持久化分区topic

  1. 校验
  2. zk中写入topic元数据：/admin/partitioned-topics/节点下写入 topic名称和分区数
  3. 创建Ledger信息：为每个分区创建Ledger信息 记录分区->bookie节点
  4. 将topic信息缓存在broker中

- 归属

  需要将topic/partition绑定到一个broker上

  设计思路：

  无限映射转化为有限映射 跟redis的slot一个思想

  先计算topic的哈希值 到哈希环中找到对应的虚拟结点bundle

  bundle与broker建立映射

  从而topic与broker建立映射

  问题：bundle与broker之间的映射怎么建立

  Lookup命令：

  1. 校验

  2. 找到该namespace下的所有bundle

  3. 计算topic所属的bundle

  4. 看对应的bundle有没有对应绑定的broker 如果没有 就需要绑定上

     绑定过程：

     1. 确定一个裁判：可能是当前broker 也可能是zk选举出来的broker leader
     2. 裁判broker通过loadManager 将负载最低的broker分配给bundle
     3. 被分配到的broker尝试在zk节点中写入的

- 迁移：

  topic流量上升 需要迁移到负载更低的broker上

  只需要更改bundle在zk的元数据

  过程：卸载 + 重新触发绑定

  卸载：

  1. 校验

  2. 将namespace和bundle的状态都置为非活动 

     所有新的消费者和生产者的连接都会被拒绝掉

     遍历并关闭bundle下绑定的topic 断开所有相关连接 和 managed-ledger

  3. 清理broker上的缓存

  4. 删除了zk中的bundle节点

- 分裂bundle：解决bundle管理的topic过多的问题

  在哈希环上添加虚拟节点

  1. 获取两个bundle范围
  2. 更新zk中namespace的bundle信息
  3. 卸载老的bundle 让topic归属到新的bundle上



**数据流客户端PulsarClient中的属性**

- lookupservice：发现broker
- connectionPool：TCP连接池化 避免对一个broker产生大量重复的连接
- ID生成器：为消息生产id 避免重复消费



**生产者客户端**

参数：

- 访问topic的模式
  - shared：默认 多个producer可以给一个topic生产消息
  - exclusive：一个topic只能由一个producer生产消息 可以保证消息顺序
  - waitForExclusive：exclusive的主备版

- 有分区的topic的时候 消息的路由方式：
  - singlePartition：绑定单一partition 消息在单一分区内有序
  - RoundRobinPartition：轮询
  - CustomPartition：自定义



**消息的发送**

都是异步 同步也是通过future.get 异步转同步

消息先被放到发送队列中 发送后从发送队列移动到等待队列 获得broker响应后 才将消息删除



**消息的顺序**：生产者发送的顺序与Broker中消息的顺序一致

- 业务线程：多个线程写一个生产者 会导致无序
- 路由模式：roundRobinPartition 多分区下无序
- topic分区：单分区下 broker对某个消息持久化失败 导致该消息重发 也会导致无序
- 发送方式：异步发送会导致无序
- 批量发送：不开启会导致无序
- 消息是否有key：无key+默认的roundRobin路由会导致无序



**保证消息只被消费一次**

同一条消息 重试的时候sequenceId不会变 

broker会拒绝sequenceId小的消息



**消息可靠性**

发送：同步和异步发送都有返回 producer能感知到消息持久化失败



**消费者客户端**

**订阅类型**

- Exclusive：默认

  多个消费者使用一个订阅名称（同一消费组）

  一个消费组下只能有一个消费者消费

  消息确认方式可以选单个和累积

- Shared：

  一个订阅组下的消费者可以共同消费消息 一个消息只能被一个消费者消费

- Failover：

  其他同Exclusive 当主消费者断开连接时 所有消息都会传递给下一个消费者

- Key_Shared：

  同一个订阅组下 所有消费者可以同时消费 不同的消费者消费不同的key对应的消息



**接收消息的步骤**

1. 根据topic 发送lookup请求 获取对应Broker

2. 从连接池中获取broker对应连接

3. 向broker发送订阅命令

4. 向broker发送**FlowPermits**命令（这个设计挺好的）

   其中包含自己还可以消费的最大窗口数 类似于TCP的接收窗口

   当接收队列中容量剩余超过一半时会重新发flowPermits

   这样可以避免消费能力不足 消息又可以及时推送

5. broker推送消息给consumer



**消费者消费消息后对消息确认异常的处理**

- 收到了多条消息 只确认了其中几条

- 获取消息后 一直不确认

  消费者调用了receive方法后 会在某个时间段后 让broker重投递该消息

  broker将该消息从已发送待确认集合中移除 该消息可以重新被消费

- 暂时消费不了 想过一会儿再消费 

  延迟队列

  延迟消息是带有延迟投递信息的消息 也是推送到业务topic中

  Broker中会为该topic维护一个优先级队列 到期时间小的在堆顶（定时器基本上都是这个实现）

  消费者消费的时候 会到堆顶看是否有到期的消息 有则消费

- 消费失败了或者很多次消费都失败了

  重试队列：也是一个普通的topic 开启重试队列后 消费者会在订阅topic时 顺便把topic对应的重试topic也订阅了

  死信队列：也是一个普通的topic 当消费者感知到某个消息的重试次数超过限制后 不会消费 而是将消息推送到死信队列对应的topic中

- 当前消费者消费不了 但可以让其他消费者试

  对某个消息发送nack



**客户端的其他能力**

- 连接管理

  一个PulsarClient中管理了一个连接池和两个线程池

  一个Broker一个TCP连接 Reactor模型

- 线程池：

  设计：维护了多个最大线程数量=1的线程池 通过计算对象的hashCode来判定分配任务到哪个线程池中执行

- LookupService

  目的：

  - broker服务的动态发现
  
    确定topic在哪个broker上 实现：重定向（集群级  broker级：topic -> bundle(一致性哈希虚拟结点) -> broker）
  
  - 元数据查询



## Broker

**接收到的生产消息时**

1. 获取broker中的producer对象（元数据）

2. 校验：broker级的待确认消息阈值 生产者级的限流 是否为事务消息 sequenceId 加密与校验和

3. 非持久化消息：直接推送 直接遍历所有订阅的consumer 发送出去 内存中也不会保存

   持久化消息：校验topic状态 消息大小 sequenceId

1. 将消息异步持久化



**接收到消息推送消息时**：consumer主动告诉broker可以推送以及可以推送的数量

1. 获取consumer对象
2. 校验：如果有很多已发送待确认的消息->限流 
3. 根据订阅模式 选择consumer
4. 过滤消息：重复 已确认的 没有到期的延迟消息 
5. 发送消息给consumer



**Schema**：确定消息的序列化和反序列化方式 有版本号 升级版本时会校验是否兼容老版本





