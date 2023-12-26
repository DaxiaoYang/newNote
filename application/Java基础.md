# Java基础

[TOC]





### 引用类型

Java是值传递的例子

```java
Student stu = new Student(1,1);
fb(stu);
System.out.println(stu.age); // 1

private static void fb(Student vstu) {
  	// 只是获取了一份内存地址的copy
  	// 无法改变引用变量的值：即指向新的地址（C++的引用传递实际上是指针的语法糖）
  	vstu = new Student(2, 2);
}
```





## 字符

**字符集**：一组字符的集合，包含字符 以及对应的编号  Unicode是字符集

**字符编码**：计算机存储字符编号的格式  UTF-8/16/32是字符编码

Unicode字符编码：

- UTF32：定长编码

  4B 

  编码：直接将编号存入

  解码：4B解码为一个字符

- UTF16：变长编码（Java编解码方式）

  2B 4B  根据一个B前面的bit固定位数

  可以判断出这个字节是2字节

  还是4字节的高位和地位

- UTF8：变长编码

  1 2 3 4字节编码都有

  也是根据一个B前面的bit 判断是几字节编码

  比如如果是3字节编码 就继续往后读2字节

  



## 字符串

**JDK6中String内存泄漏问题**：

```java
String s = "abcde";
String substr = s.substring(1, 4); // 引用的同一个value数组 只不过offset和count不同
// 只有当s和substr都不被引用了 value数组才能释放
```



**JDK9中的压缩技术**：也是经典的变长存储思想 根据数据表示范围的大小 选择最小的存储空间（序列化协议中也常见）

```java
// 采用字节数组存储字符
byte[] value;
// latin1 / utf16 分析字符串决定 只有ASCII字符 coder=latin1 还有其他字符就还是原来的utf16
byte coder;
```



**String的常量池技术**

字符串常量池：需要一个map 在创建String的时候 判断String对象是否已经存在常量池中

- JDK6：PermGen永久代（空间有限）
- JDK7：堆中

```java
// 放入常量池
String a = "aaa";
// 不放入常量池
String b = new String("aaa"); // b还是指向原来堆中的对象 b = null 时 就会被回收掉
String c = b.intern(); // c == a 引用相同
// 使用场景：无法直接通过字符串常量来赋值 如用api读取数据库 有大量重复字符串 就可以用intern()来复用内存空间
```



**String的不可变性**

- 常量池技术下：会有多处引用同一个字符数组 
- 字符串和整型会被经常用来做HashMap的key value数组可变的话 对应的hash值也会变 在HashMap中的位置也会变



## 对象

关注对象内存大小的原因：JVM内存中只有堆的内存比较值得关注，而堆中都是对象占的内存



结构：

- 对象头：

  标记字：8B（用于GC和多线程）

  类指针：8B（压缩后4B）

  数组长度：4B（所以Java中数组最大长度为2^32）

- 实例数据：不会按定义的先后顺序来存储 会在满足对齐的前提下 尽可能压缩空间
- 对齐填充：按8字节对齐（JVM参数可调）

```xml
<dependency>
         <groupId>org.openjdk.jol</groupId>
            <artifactId>jol-core</artifactId>
            <version>0.16</version>
        </dependency>
```



```java
class A {
    int a;
    float b;
}

class B extends A {
    double c;
    Integer d;
    int e;
    static int A;
}
// 8字节对齐
System.out.println(ClassLayout.parseInstance(new B()).toPrintable());
B object internals:
OFF  SZ                TYPE DESCRIPTION               VALUE
  0   8                     (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4                     (object header: class)    0x00060c18
 12   4                 int A.a                       0
 16   4               float A.b                       0.0
 20   4                 int B.e                       0
 24   8              double B.c                       0.0
 32   4   java.lang.Integer B.d                       null
 36   4                     (object alignment gap)    
Instance size: 40 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total


// 数组对象会多4B的数组长度
System.out.println(ClassLayout.parseInstance(new int[]{1}).toPrintable());

OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0x00000b98
 12   4        (array length)            1
 12   4        (alignment/padding gap)   
 16   4    int [I.<elements>             N/A
 20   4        (object alignment gap) 
```



**压缩类指针和引用**

64位机器中，引用是8B，占用的范围太大，而一般进程的内存没有那么大，不需要这么多寻址空间

所以为了节省内存 用4B存储对象引用

又因为单个对象都是8字节对齐的 所以对象的地址都是8的整数倍 后3位为0   `1000`

所以存储的时候 不存储后3位 这样可以表示的范围就是2^35B  -> 32B（设置的JVM堆大小超过32GB时 压缩引用不会生效 就还是8B）

如果还需要再扩大表示范围 可以通过增大对象对齐字节的单位(16 32... 2的幂都行)



## 关键字

**final**

- final还可以修饰方法参数

- 不可变类实现：

  - final class：禁止继承 覆写方法可以暴露属性
  - final 属性：只在初始化时赋值一次

- lambda/匿名内部类只能接受final/实际final变量的原因：Java都是值传递  修改了之后 外面的变量没有改变 不符合直觉

  ```java
  int a = 1;
  // 假如可以更改值
  Thread thread = new Thread(() -> {
    System.out.println("start");
    a = 3; // 这里会报错
  });
  thread.start();
  thread.join();
  // 这里打印的还是a = 1
  System.out.println(a);
  
  
  public void localVariableMultithreading() {
    	// 值传递 只是个copy非常非常疑惑人
      boolean run = true;
      executor.execute(() -> {
          while (run) {
              // do operation
          }
      });
      
      run = false;
  }
  ```

  



**static**

- 静态变量：跟类的代码一起存储在方法区

- 静态方法：只能访问静态变量/方法

  对象可以使用类的数据 类不能使用某个具体的对象的数据

- 静态内部类：跟静态方法一样 只能访问外部类的静态变量 自身可以包含静态变量和方法



```java
public class Singleton { 

    static {
        System.out.println("singleton class load");
    }

    private Singleton() {}

    private static class SingletonHolder {
        static final Singleton INSTANCE = new Singleton();
        static {
           System.out.println("SingletonHolder class load");
        }
    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }

    public static void main(String[] args) {
        Singleton.getInstance();
    }
}

// singleton class load  调用了静态方法触发类加载
// SingletonHolder class load 调用静态变量触发类加载
```

问题：

- 如何保证线程安全

  JVM加载SingletonHolder类时保证（类加载到方法区 类加载过程是线程安全的）

- 如何保证延迟加载

  Java中的类都是延迟加载 在用到的时候才会触发

- SingletonHolder为什么是静态内部类 是否可以是普通内部类

  不可用 普通内部类中不能声明静态变量

- 为什么SingletonHolder的访问权限设置为private

  因为只有外部类的代码访问

 

## 容器工具类

sort()函数：

- 会将集合中的元素放到数组中再进行排序 最终调用的都是Arrays.sort

- 对于基本数据类型 采用不稳定的DualPivotQuicksort

  两个pivot的快速排序

- 对于对象类型 采用稳定的TimSort

  非递归的归并排序 + 二分插入排序



binarySearch()函数：

- 对于支持随机访问的列表 采用下标
- 对于不支持随机访问的列表 采用迭代器
  - 元素个数<5000 还是普通二分
  - 元素格式 >= 5000 getMid优化 时间复杂度O(n)



### LinkedHashMap

结构：哈希表+双向链表

实现按插入顺序遍历的方式：插入哈希表的同时，将节点链入链表表尾，迭代器遍历从双向链表表头开始

LRU实现：

- 支持O(1)增删改查：哈希表
- 缓存满的时候支持O(1)内找到最近最久未访问的元素并删除：双向链表



LinkedHashMap实现LRU：

- accessOrder：true 按访问顺序 将元素放到末尾

- 重写removeEldestEntry方法：size() > 自定义缓存的大小



### NIO类库：BIO NIO AIO

阻塞与非阻塞：

- 阻塞：read的时候 如果数据没有到达用户缓冲区 就一直阻塞等待 

  去饭店一直等待叫号

- 非阻塞：read的时候 如果数据没有到达用户缓冲区 就直接返回false 

  去饭店领号后 先去逛商场 时不时过来问有没有到

同步与异步：

- 同步：read的时候 主要线程去主动阻塞等待/轮询

  需要自己去问有没有到号

- 异步：read的时候 注册回调事件 让操作系统执行回调 

  饭店消息通知你已经到号了





**Java不同IO模型实现服务器开发**

- BIO

  连接+每个处理读事件 都阻塞

  多线程方案

- NIO

  IO多路复用 连接阻塞 一个线程可以处理多个读事件

- AIO

  注册连接完成事件 注册可读事件



### 高速IO（上）：普通IO读写流程都存在哪些性能问题

用户态->内核态的上下文切换耗时组成：

- 寄存器内容的保存与恢复

  栈指针：内核空间有自己的函数调用栈

  程序计数器：执行内核代码

- CPU缓存失效

  用户程序的指令和数据缓存都失效了（不相邻） 都得从内存重新加载



![image-20231205192249043](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202312052114222.png)

IO读写流程：

- 读：

  应用程序执行read系统调用

  操作系统先检查内核读缓冲区有没有（每个文件描述符都有其对应的 读缓冲区和写缓冲区）

  有就直接拷贝 没有就发起IO请求到DMAC 从IO设备读取哪些数据到哪块内核缓冲区内存  DMAC拷贝完成之后 通过中断通知CPU 

  然后CPU再将内核读缓冲区的数据拷贝到应用程序缓冲区

- 写：

  应用程序执行write系统调用

  CPU将应用程序缓冲区中的数据 拷贝到内核缓冲区（拷贝完成之后write就返回了）

  除非手动调用sync 否则是操作系统自己写盘 写盘的过程也是DMAC负责



### 高速IO（下）：mmap和零拷贝是如何提高IO读写速度的

手段：减少数据拷贝 减少上下文切换



**mmap**：针对文件读写，系统调用mmap（mmap还可以通过共享物理内存 来实现进程间通信）

步骤：

1. 程序的用户虚拟内存和磁盘上的文件（或文件中的某段）建立映射
2. 读取虚拟内存的时候 如果发现缺页 就加载文件到物理内存中（只拷贝一次 普通的IO需要两次拷贝 IO设备->内核缓冲区->用户缓冲区）
3. 修改虚拟内存中的数据 操作系统后续会将脏页刷回磁盘（也可以主动调用msync）

![image-20231206085931502](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202312060859575.png)

```c++
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/fcntl.h>
#include <sys/mman.h>

void forkDemo();

int main() {
//    forkDemo();
    char* file = "/Users/yangsiping/Downloads/027主分支.log";
    int fd = open(file, O_RDWR, 0666);
    if (fd < 0) {
        printf("open file filed");
        return -1;
    }
    size_t len = 64;
    int offset = 0;
    // 映射文件0-63字节 到 虚拟内存
    char* ptr = static_cast<char *>(mmap(nullptr, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, offset));
    if (ptr == MAP_FAILED) {
        printf("mmap failed");
        return -1;
    }
    close(fd);
    // 操作ptr就相当于在读写文件
    for (int i = 0; i < len; i++) {
        ptr[i] = '0';
    }
    for (int i = 0; i < 512; i++) {
        printf("%c", ptr[i]);
    }
    munmap(ptr, len);
    return 0;
}
```



**零拷贝**：针对网络传输，系统调用sendfile

步骤：

1. 从源IO设备拷贝数据到内核读缓冲区（DMA）
2. 内核读缓冲区拷贝另一个IO设备的内核写缓冲区（CPU拷贝）
3. 内核写缓冲区拷贝到目的IO设备(DMA)

![image-20231206090022369](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202312060900399.png)



## JVM

### 编译执行

Java编译流程图

![image-20231207082806218](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202312070828284.png)

前端编译：Java文件->class文件

- 注解处理 lombok
- 解语法糖 泛型 for-each loop



类的字节码格式：

- 元信息
- 常量池
- 字段表
- 方法表

```json
// javap -c CompileDemo.class 
Last modified 2023-12-7; size 349 bytes
  MD5 checksum d0d1ae32a24a964539fdb3f07abf6f96
  Compiled from "CompileDemo.java"
public class CompileDemo
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#17         // java/lang/Object."<init>":()V
   #2 = String             #18            // daxiao
   #3 = Fieldref           #5.#19         // CompileDemo.name:Ljava/lang/String;
   #4 = String             #20            // hello
   #5 = Class              #21            // CompileDemo
   #6 = Class              #22            // java/lang/Object
   #7 = Utf8               name
   #8 = Utf8               Ljava/lang/String;
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               sayHell
  #14 = Utf8               ()Ljava/lang/String;
  #15 = Utf8               SourceFile
  #16 = Utf8               CompileDemo.java
  #17 = NameAndType        #9:#10         // "<init>":()V
  #18 = Utf8               daxiao
  #19 = NameAndType        #7:#8          // name:Ljava/lang/String;
  #20 = Utf8               hello
  #21 = Utf8               CompileDemo
  #22 = Utf8               java/lang/Object
{
  public CompileDemo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #2                  // String daxiao
         7: putfield      #3                  // Field name:Ljava/lang/String;
        10: return
      LineNumberTable:
        line 6: 0
        line 8: 4
}
SourceFile: "CompileDemo.java"

```



解释执行例子：

```java
public class App {
  	public static void main(String[] args) {
      	Demo demo = new Demo();
      	demo.sayHello();
    }
}
```

1. 执行到Demo对象实例化：发现Demo类没有加载，执行类加载，将Demo.class加载到内存方法区，并根据类的字节码在堆中创建对象
2. 执行对象方法：根据对象的类指针，找到方法区Demo类，然后在类的方法表中查找sayHello函数对应的字节码 然后逐句解释执行

![image-20231207091725332](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202312070917431.png)



JIT与AOT:

- JIT：将热点代码的字节码在运行时直接编译为机器码（因为在运行时进行 所以可以收集运行时的信息 进行相应的代码优化）

  热点探测：探测到频繁执行的代码（会有热度衰减 过一段时间 如果次数没有达到阈值 就会把计数减半）探测的对象有方法和循环代码

  编译优化：

  - 方法内联：方法执行次数够多 + 方法字节码够小 本质也是空间换时间 因为展开之后 程序会占更多的内存 但是节省了入栈出栈的时间 （final方法 会节省JVM分析的时间 但是内不内联还是得看被执行次数+大小）
  - 逃逸分析：单线程+方法内
    - 栈上分配：对象如果生命周期只在方法内 那可以直接在栈上分配内存 这样对象内存回收更快 减少GC压力
    - 标量替换：对象生命周期只在方法内 直接用基本类型 代替对象属性
    - 锁消除：锁的范围 只有方法内 其他地方不会访问 会优化掉锁的代码

- AOT：一步到位 在运行前 直接将源码编译为机器码



### 类加载

**双亲委派加载**：

当需要加载某个类时，应用程序类加载器问扩展加载器要，扩展加载器问起始加载器要，启动加载器在自己的目录下找不到，让扩展类加载器在自己的目录下找，启动类加载器在自己的目录下找不到，让应用类加载器在自己负责的目录下找

这样的好处是 可以避免核心类被篡改的现象



**自定义类加载器**

重写findClass方法，在自己的目录下 尝试寻找类对应的class文件，找到了之后将其读入内存，转化为Class对象（信息存储在方法区中）

> class文件解密就可以在类加载的时候做



### 内存分区

线程私有：

- 程序计数器：

  记录当前线程要执行的下一条指令（虚拟机指令）

  跟CPU的PC的关系是 CPU的PC是多个线程共享的，线程切换时，需要重新设置PC

- 虚拟机栈：-Xss参数调整线程栈的大小

- 本地方法栈（Java调用native方法时 native方法的栈 在hotspot具体实现中 跟虚拟机栈放一个栈里面）

线程共享：

- 堆

  年轻代

  老年代

- 方法区

  - 类信息
  - 方法信息
  - 静态变量
  - 运行时常量池（每个类对应一个 存储符号引用）
  - 字符串常量池：String对象，类之间共享
  - JIT编译后的机器码



### 可达性分析

**GC Roots**：

- 内容：

  堆外变量上的引用直接指向的对象，堆外变量指栈上的局部变量和方法区中的静态变量

- 维护GC Roots：

  使用OopMap来存储动态变化的GC Roots

  先初始化OopMap 然后在代码执行过程中 如果有变量更新引用的对象，JVM就同步更新OopMaps

- 安全点/安全区

  线程代码执行到这里时， 维护GC Roots的OopMaps才是个有效的状态



### 垃圾回收算法

方法区中的回收：

- 无用的String对象：

  存储在字符串常量池中 并且没有被变量引用的String对象

- 无用的类：

  - 类的所有对象都被回收
  - 类的Class对象没有被变量引用
  - 加载该类的类加载器已经卸载



进入老年代规则：

- 固定GC年龄：默认值为15

- 年轻代中年龄较大的对象超过了一定的占比，如50%，就把这些年龄大的对象都放入老年代（提前放）

  40% GC age = 1 60% GC age = 5 那就直接把GC age = 5的对象放到老年代



为什么FullGC比YoungGC要慢

- 遍历的范围更大：老年代 年轻代 元空间
- 存活的对象更多：Young GC中 年轻代存活的对象少 遍历快，Full GC中 存活的对象多，遍历慢



空间担保：

年轻代：分为1个Eden区和2个Survivor区，当to Survivor区的空间不够时，会将对象放到老年代，如果老年代的空间不够，那就进行FullGC，如果FullGC之后空间还是不够，那就OOM





### 垃圾回收器选型

**垃圾回收器评判标准**：

- 吞吐量：

  进程时间 = 用户线程时间 + GC线程时间

  用户线程时间占比越高 吞吐量越大

- 停顿时间

  GC所需暂停用户线程的时间 越短越好

- 资源消耗

  GC所需耗费的CPU和内存 

  

**常用的垃圾回收器**

- Serial

  单线程 回收的时候不能并行 需要停顿用户线程

  分类：

  - Serial New：用于年轻代 标记复制算法（适用于存活对象比较少的时候 只需移动存活对象 对象复制量最少）
  - Serial Old：用于老年代 标记整理算法（适用于存活对象比较多的情况 对象复制量最小）

  适用于单核场景

- Parallel

  多线程 回收的时候不能并行 需要停顿用户线程

  分类：

  Parallel Scavenge和Parallel New：年轻代 标记复制

  Parallel Old：老年代 标记整理

  适用于需要吞吐量大的场景

- CMS

  多线程 回收的时候 有些阶段可以并行 

  用于老年代 标记清理 多次清理之后 跟一次整理 避免过多内存碎片

  适用于需要STW时间少的场景

- G1（设计很值得借鉴）

  分批处理 化整为零的思想

  分区 每个区属于年轻代（Eden 或者 Survivor） 或老年代

  多线程 年轻代使用标记复制 老年代使用标记清除

  每次垃圾回收的范围可以设置 使得STW时间不会过长

  CMS高级版 适用于需要STW时间少的场景

![image-20231213202604363](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202312132026489.png)



### 垃圾回收器实现：CMS和G1的并发垃圾回收

**并发垃圾回收**：CMS和G1都是这么做的

1. 初始标记：标记出GC Roots（需要停顿用户线程）

2. 并发标记：

   **可达性分析实现：三色标记法**（BFS/DFS）

   颜色含义：

   - 白色：未遍历到的对象

   - 灰色：对象自身被遍历，但是对象直接引用的对象未被遍历

   - 黑色：对象自身被遍历，且对象直接引用的对象也被遍历

   遍历结束后 如果对象为白色的 那就说明该对象不可达

   具体标记步骤：

   1. 将GC Roots中的所有对象标记为灰色，放入灰色集合，其余对象标记为白色，放入白色集合
   2. 依次将灰色集合中的对象标记为黑色，放入黑色集合，将其直接引用的对象标记为灰色，放入灰色集合
   3. 重复执行步骤2 到最后只剩黑色集合（可达对象）和白色集合（不可达对象）

   ![image-20231214075828566](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202312140758621.png)

   因为并发标记时，应用程序也在运行，所以可能改变对象的引用关系，出现误标和漏标的情况

   > 误标可以容忍：逃过GC的不可达对象可以在下一次GC中清理
   >
   > 上图步骤3中 
   >
   > ```java
   > objA.fieldB = null
   > ```
   >
   > 这样B这个不可达对象（及其引用的对象）就会被认为是可达的 逃过一次GC
   >
   > 
   >
   > 漏标不能容忍 ：会回收掉存活的对象 导致应用程序出错 
   >
   > 上图步骤3中
   >
   > ```java
   > // 新增引用：黑色对象 -> 白色对象
   > objA.fieldC = objA.fieldB.fieldC
   > // 删除引用: 灰色对象 -> 白色对象
   > objA.fieldB.fieldC = null
   > ```
   >
   > 这样的话 C会被认为不可达 漏标为可达性对象 会被回收

3. 重新标记：对漏标进行修正（根据并发标记中记录的应用程序对引用的修改 进行批量处理）

   - 增量更新：记录新增引用关系 CMS方案

     并发标记中 如果应用程序新增了一个黑色->白色的引用关系

     记录下来 重新标记的时候会以这些白色对象为起点 重新进行可达性分析

     这样漏标的白色对象 会被标记为黑色对象

   - 原始快照：记录删除引用关系 G1方案

     并发标记中 如果应用程序删除了 灰色->白色（直接/间接引用）的引用关系

     重新标记的时候 会以这些白色对象为初始点重新遍历 这样会导致误报（不可达误报为可达） 但是可以容忍

4. 并发清除

   清理重新标记中 标记出来的不可达对象

   可达 -> 不可达（可能发生 那就会误报）

   不可达 -> 可达（不可能发生 因为从堆外变量中根本引用不到这个对象了）

   新创建的对象 标记为黑色（这轮先不处理）

   CMS和G1都使用标记清除算法：是考虑到时间-空间的权衡 多次清理后 整理一次 时间和空间都能得到兼顾









### JVM性能调优

关注的东西：

GC频率和时间（特别是fullGC）

- 年轻代对象增长速率
- youngGC后存活对象的大小
- youngGC后进入老年代对象大小
- 老年代对象的增长速率



### JVM问题排查

**JVM性能分析工具**

- GC统计信息监控

  ```shell
  jstat -gcutil pid intervalMs
  ```

- GC详细日志分析：不同垃圾回收器日志不一样

  ```
  -XX:+PrintGCDetails
  ```

- 获取内存快照

  ```shell
  -XX:HeapDumpxxx # 在FullGC或者OOM前 输出堆
  
  jmap -dump:format=b,file=fileName pid # 输出进程的堆内存快照
  ```



**JVM常见问题**

- OOM

  诱因：

  - 设置堆/元空间的大小太小

  - 一次性创建过多的对象：SQL查询全表数据

  - 内存泄漏： 分析hprof文件

    ```shell
    java -Xms100m -Xmx100m -Xmn60m -XX:SurvivorRatio=8 -XX:+UseConcMarkSweepGC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/yangsiping/programme/Java/javabase/ jvm.OomDemo
    ```

- 频繁GC

  频繁YoungGC：增大年轻代大小

  频繁FullGC：

  - 内存泄漏
  - 大对象生成
  - 空间担保分配：to Survivor不足 GC日志里面有

- GC时间过长

  - 堆内存过大

    选G1 或者减少JVM内存 多实例部署

  - Concurrent Mode Failure

    给垃圾回收器自身准备的预留内存空间过小

  - 操作系统swap

    JVM的内存 不要超过操作系统的内存 并且要预留出空间来



