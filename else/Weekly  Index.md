# Weekly 

## 2021







阶段一：（7天）热量≈846kcal/天，成本≈45/天，体重下降6-8斤建议孕产妇，抑郁症和晚期肿瘤患者，身体过度消瘦、营养不良的人，年龄在 18 岁以下、70岁以上，患有严重心脑血管、精神障碍、慢性感染疾病的人群等不宜贸然尝试早餐：小轻砖*2（170kcal）+伊利舒化奶*1（142kcal）中餐：小轻砖*2（170kcal）+核桃*3颗（126kcal）*1+小份水果（80kcal）晚餐：小轻砖*1（85kcal）+鸡蛋*1（73kcal）先插一下，食谱中所有食物购买渠道：超市：核桃、水果、鸡蛋、酸奶京东：伊利舒化奶，很便宜Miniso（名创优品）：卤牛肉未来可7旗舰店：鸡胸肉、小轻砖、全麦吐司

阶段二：限能量平衡膳食CRD（21天）热量≈1228kcal，成本≈60/天，体重下降12-15斤早餐：小轻砖*2（170kcal）+鸡蛋*1（73kcal）+伊利舒化奶*1（142kcal）中餐：小轻砖*2 （170kcal）+香蕉*1（115kcal）+酸奶*1（141kcal）+鸡胸肉40g（54kcal）晚餐：全麦吐司*1（109kcal）+卤牛肉75g（225kcal）+黄瓜1根（29kcal）注：此阶段请开始有氧运动，建议大家从跑步开始尝试，我是从2km开始的，到后面每天都可以跑5-7km。

阶段三：维稳（3个月）热量≈1600kcal/天，成本≈53/天，体重下降8-10斤早餐：小轻砖*2（170kcal）+鸡蛋*1（73kcal）+伊利舒化奶*1（142kcal）中餐：少油少盐多蔬菜的食堂餐（1000kcal以内）晚餐：小轻砖2（170kcal）+黄瓜*1（29kcal）



### 5.10-5.16

- [x] Leetcode 7

- [ ] 设计模式21

- [ ] 计组10篇

- [ ] 计组pdf25pages

- [ ] JUC两个点

  `CurrentHashMap`

  `ThreadLocal`

- [ ] Docker

- [ ] Spring循环依赖解决

- [ ] Java序列化与反序列化

- [x] `copy on writearrayset`

  `add`方法时加锁的 往数组中添加一个元素的时候 先创建一个新的数组 往新数组中添加元素 最后替换引用 使得数组指针指向新对象 然后解锁

  `get`方法直接获取当前对象的数组

- [ ] [简易RPC实现](https://github.com/wangzheng0822/codedesign/tree/master/com/xzg/cd/rpc)

- [ ] 幂等

- [ ] 对称性加密和非对称性加密 李永乐视频

  > Your understanding of "public keys encrypt, private keys decrypt" is correct... for data/message ENCRYPTION. For digital signatures, it is the reverse. With a digital signature, you are trying to prove that the document signed by you came from you. To do that, you need to use something that only YOU have: your private key.
  >
  > A digital signature in its simplest description is a hash (SHA1, MD5, etc.) of the data (file, message, etc.) that is subsequently encrypted with the signer's private key. Since that is something only the signer has (or should have) that is where the trust comes from. EVERYONE has (or should have) access to the signer's public key.
  >
  > So, to validate a digital signature, the recipient
  >
  > 1. Calculates a hash of the same data (file, message, etc.),
  > 2. Decrypts the digital signature using the sender's PUBLIC key, and
  > 3. Compares the 2 hash values.
  >
  > If they match, the signature is considered valid. If they don't match, it either means that a different key was used to sign it, or that the data has been altered (either intentionally or unintentionally).

  

- [x] [JWT introduction](https://jwt.io/introduction) 

- [x] RSA公钥加密为什么每次生成的都不一样 ：[原因](https://blog.csdn.net/guyongqiangx/article/details/74930951)

  + 公钥私钥加密前都需要对数据进行填充 
    + 私钥：填充的某个部分的值为固定的值
    + 公钥：填充的某个部分的值为伪随机数



+ [x] `java spi`

  [spi in sprintboot-starter](https://juejin.cn/post/6844903890173837326)

  `serviceLoader`通过反射创建实例

+ [ ] `swagger`

+ [ ] `Disruptor` 下个月再看

+ [x] cookie是存储在哪的

  存储在文件中

+ [ ] 线程池`shutdown`方法

+ [x] gc roots的对象有哪些

  + 与栈帧相关的各种对象

  + 当前被加载的Java类

  + Java 类的引用类型静态变量。

  + 运行时常量池里的引用类型常量（String 或 Class 类型）。

  + JVM 内部数据结构的一些引用，比如 sun.jvm.hotspot.memory.Universe 类。

  + 用于同步的监控对象，比如调用了对象的 wait() 方法。

  + JNI handles，包括 global handles 和 local handles

+ [ ] 压缩指针

+ [x] [强引用 弱引用 软引用 WeakReference weakhashmap]( https://www.baeldung.com/java-weakhashmap)

  `weakreference`: 

  > When an object in memory is reachable only by Weak Reference Objects, it becomes automatically eligible for GC.

  `strong reference`

  > That is, whenever an object is referenced by a *chain of strong Reference Objects*, it cannot be garbage collected.

- [ ] CDN
- [ ] `google guava eventbus`
- [ ] 有限状态机
- [ ] `event bus`实现





### 5.17-5.23

+ [x] leetcode7+

+ [x] design pattern 21+

+ [ ] computer organization 14

+ [x] 复习一下最小接口原则

+ [ ] 敏捷开发

+ [x] Volatile to reference type

  `volatile` acts either on a primitive variable or on a reference variable. There’s no relation between `volatile` and the object referred to by the (reference) variable.

+ [ ] `simple dateformat` 线程安全问题 看下源码

+ [x] uml图

+ [ ] 线程被中断后执行

+ [ ] https://redis.io/topics/persistence

+ [x] 字符串匹配算法

+ [x] 看下模板和策略模式

+ [x] 看下lambda那块的内容

+ [x] `protocol buffer`

+ [x] `? extends ? super`复习一下这个



### 5.24-5.30

- [ ] Leetcode 7

- [ ] design pattern 20

- [ ] 计组 7

- [ ] `event bus`源码

- [x] `lambda`表达式实现：内部类 是一个语法糖

- [x] *little endian* 字节序

  `0x01234567`

  ![](http://4.bp.blogspot.com/_IEmaCFe3y9g/SO3GGEF4UkI/AAAAAAAAAAc/z7waF2Lwg0s/s400/lb.GIF)

- [ ] 为什么栈是从高位向地位扩展的

- [ ] [`disruptor`](https://lmax-exchange.github.io/disruptor/disruptor.html)

- [x] `FileChannel.map -> MappedByteBuffer `  

  可以加载文件的部分到内存中

  ```java
  public void givenFile_whenReadAFileSectionIntoMemoryWithFileChannel_thenCorrect() 
    throws IOException { 
      try (RandomAccessFile reader = new RandomAccessFile("src/test/resources/test_read.in", "r");
          FileChannel channel = reader.getChannel();
          ByteArrayOutputStream out = new ByteArrayOutputStream()) {
  
          MappedByteBuffer buff = channel.map(FileChannel.MapMode.READ_ONLY, 6, 5);
  
          if(buff.hasRemaining()) {
            byte[] data = new byte[buff.remaining()];
            buff.get(data);
            assertEquals("world", new String(data, StandardCharsets.UTF_8));	
          }
      }
  }
  ```

  [FileChannel](https://www.baeldung.com/java-filechannel)

  


### 6.7-6.13

Todo List

- [ ] Leetcode 7

- [ ] 计组 7

- [ ] Tomcat & Jetty 14

  

Problem Set:

- `spring`单例的范围是IOC容器

- `memory swapping`

- 再看一下算法专栏hash的用途

- **布隆过滤器: java: BitSet Guava bloomFilter**

- `gitub`的使用

- 再梳理一下函数调用的过程 汇编

- [`java.nio`](http://tutorials.jenkov.com/java-nio/index.html)

  > The meaing of non-blocking IO:
  >
  > Java IO's various streams are blocking. That means, that when a thread invokes a `read()` or `write()`, that thread is blocked until there is some data to read, or the data is fully written. The thread can do nothing else in the meantime.
  >
  > Java NIO's non-blocking mode enables a thread to request reading data from a channel, and only get what is currently available, or nothing at all, if no data is currently available. Rather than remain blocked until data becomes available for reading, the thread can go on with something else.
  >
  > The same is true for non-blocking writing. A thread can request that some data be written to a channel, but not wait for it to be fully written. The thread can then go on and do something else in the mean time.
  >
  > What threads spend their idle time on when not blocked in IO calls, is usually performing IO on other channels in the meantime. That is, a single thread can now manage multiple channels of input and output.

- [tomcat热部署](https://blog.csdn.net/joeyon1985/article/details/46722655)

- ==shell脚本编程==

- ==clickhouse==

- servlet规范

- [UML类图](https://zhuanlan.zhihu.com/p/109655171)

- [ ] [Java nio server](https://github.com/jjenkov/java-nio-server)

- [ ] [WebSocket](https://www.tutorialspoint.com/websockets/index.htm)

- [ ] `unix select poll epoll`

- [ ] [Java classpath](https://stackoverflow.com/questions/2396493/what-is-a-classpath-and-how-do-i-set-it)







- [x] kafka入门
- [x] pulsar入门
- [ ] clickhouse入门
- [ ] AQS源码





Msgpack用法 https://github.com/msgpack/msgpack-java/blob/develop/msgpack-jackson/README.md



### 7.12

- [ ] ThreadLocalRandom()
- [ ] abelian group
- [ ] 补码加法出现溢出的情况
- [ ] `kafka zero copy`





推荐CMU在YouTube上的[Intro/Advanced] Database System 

CMU 15-445



### 8.9-8.15

- [ ] leetcode 7
- [ ] db4章
- [ ] clickhouse150 -> 210
- [ ] distruptor源码 每天半小时

[UUID](https://cloud.tencent.com/developer/article/1530850)

[uuid creator](https://github.com/f4b6a3/uuid-creator)

https://stackoverflow.com/questions/18244897/how-to-generate-time-based-uuids

[MMAP](https://www.cnblogs.com/huxiao-tee/p/4660352.html)





### 8.30-9.5

- [ ] leetcode 7

  - [x] 1
  - [x] 2
  - [x] 3
  - [x] 4
  - [x] 5
  - [x] 6
  - [x] 7

- [ ] spring

  - [ ] 1
  - [ ] 2
  - [ ] 3
  - [x] 4
  - [x] 5
  - [x] 6
  - [x] 7

- [ ] database 7

  - [x] 1

  - [x] 2

  - [ ] 3

  - [x] 4

  - [ ] 5

  - [ ] 6

  - [ ] 7

    

- [ ] 博客：eventbus和mergetree





### 9.6-9.12

- [ ] leetcode 7

  - [x] 1
  - [ ] 2
  - [ ] 3
  - [ ] 4
  - [ ] 5
  - [ ] 6
  - [ ] 7

- [ ] spring

  - [x] 1
  - [ ] 2
  - [ ] 3
  - [ ] 4
  - [ ] 5
  - [ ] 6
  - [ ] 7

- [ ] database 7

  - [ ] 1

  - [ ] 2

  - [ ] 3

  - [ ] 4

  - [ ] 5

  - [ ] 6

  - [ ] 7

