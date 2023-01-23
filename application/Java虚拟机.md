# Java虚拟机

[TOC]

## 基本原理

### Java代码是怎么运行的

JRE：JVM + Java核心类库

JDK：JRE + 开发和诊断工具



**为什么Java要在虚拟机上运行**

- 使得Java语言跨平台
- 托管环境：内存管理与垃圾回收以及其他如数组越界 安全权限等与业务逻辑无关的代码



**JVM如何运行字节码**

![](https://static001.geekbang.org/resource/image/ab/77/ab5c3523af08e0bf2f689c1d6033ef77.png)

虚拟机视角：

1. 将内存划分为几个区域：堆 方法区 PC寄存器 Java方法栈 本地方法栈

2. 将class文件加载到方法区中，实际运行时会执行方法区中的代码

3. 每调用一个方法就会在当前线程的Java方法栈生成一个栈帧 存放局部变量与字节码的操作数 栈帧大小提前计算好 **且不要求栈帧在内存空间中连续分布**（链式栈）

   退出方法时 弹出当前方法的栈帧



硬件视角：

无法直接执行字节码 需要将字节码翻译为机器码

翻译方式：

- 解释执行 逐条将字节码翻译为机器码
- 即时编译 运行中将一个方法中的所有字节码都编译为机器码再执行

HotSpot：默认为混合模式，先解释执行，然后将热点代码以方法为单位进行即时编译



**JVM运行效率**

即时编译优点：

可以知道程序的运行时信息 可以根据这个信息进行优化  理论上可以超过C++静态编译后的程序



HotSpot编译方式：

- 分层编译 热点方法首先会被Client编译器编译 热点方法中的热点会进一步被Server编译器编译（28原则 20%的代码 占了80%的计算资源）
- 即使编译的行为放在额外的编译线程中进行，编译完成后的机器码会在下次调用该方法时启用 替换原来的解释执行（类似于Copy-on-write的思想）



### Java的基本类型



```java
public class Foo {
  public static void main(String[] args) {
    boolean 吃过饭没 = 2; // 直接编译的话javac会报错
    if (吃过饭没) System.out.println("吃了");
    if (true == 吃过饭没) System.out.println("真吃了");
  }
}
```

boolean类型被映射为了int类型 true -> 1 false -> 0



![](https://static001.geekbang.org/resource/image/77/45/77dfb788a8ad5877e77fc28ed2d51745.png)

> 内部符号可以告知JVM类型对应的数组范围 从而执行对应的掩码操作



**Java基本类型的大小**

- 在堆中：与值域吻合 byte 1B char 2B boolean在堆中为一字节 boolean数组用byte数组实现 存储时做掩码操作 & 1 取最后一位

  将基础类型的值存入字段或者数组单元时 JVM会做相应的掩码操作 读取时会扩展为int

- 在解释执行的方法栈帧中：

  局部变量区等同于一个数组

  long和double用两个数组单元来存储 

  其余的基本类型和引用类型的值都只用一个数组单元 （一个数组单元的大小取决于HotSpot的位数）

栈帧的两个组成部分：

- 局部变量区：

  - 狭义的局部变量

  - this指针

  - 方法接收的参数

- 操作数栈

  将堆中的boolean byte char short加载到操作数栈上后 将其当成int类型来运算



```java
// 替换为2 10时无输出
// 替换为3 11时两个都输出 
// 存储到堆中的boolean时做了掩码操作
public class Foo {
  static boolean boolValue;
  public static void main(String[] args) {
    boolValue = true; // 将这个true替换为2或者3，再看看打印结果
    if (boolValue) System.out.println("Hello, Java!");
    if (boolValue == true) System.out.println("Hello, JVM!");
  }
}
```



### 类加载

盖房子：

1. 请建筑师出方案
2. 去市政部门报备验证
3. 盖完之后装修

类加载：

1. 加载
2. 链接
3. 初始化



引用类型：

- 类 （泛型参数）
- 接口
- 数组类（JVM直接生成）



**加载**

> 借助类加载器 查找字节流 并创建类



类加载器层级

- 启动类加载器 C++实现 没有对应的Java对象 其他类加载都是`java.lang.ClassLoader`的子类 都需要先由类加载器加载到JVM后才能使用
- 扩展类加载器：父类是启动类加载器
- 应用类加载器：父类是扩展类加载器
- 自定义类加载器：可以用来实现加解密class文件



类加载器实例 + 类的全名 = 类的ID



过程：双亲委派



**链接**

> 将创建的类合并到JVM中

- 验证：满足JVM的约束条件

- 准备：

  为类的静态字段分配内存

  构造方法表

- 解析：符号引用 -> 实际引用 （符号引用会触发未加载类的加载）

  

**初始化**

- 只会执行一次（且是JVM保证线程安全 所以可以用内部类的类加载来实现懒加载）

- 为标记为常量值的字段赋值
- 执行`<clinit>`方法（方法里执行其他直接赋值操作 以及静态代码块 通过加锁保证线程安全） 



类初始化的触发时机：

- JVM启动的主类
- new对象时
- 访问类的静态方法或字段
- 其子类初始化时
- 反射调用
- MethodHandle



### 方法调用

```java
void invoke(Object obj, Object... args) { ... }
void invoke(String s, Object obj, Object... args) { ... }

invoke(null, 1);    // 调用第二个invoke方法
invoke(null, 1, 2); // 调用第二个invoke方法
invoke(null, new Object[]{1}); 
// 只有手动绕开可变长参数的语法糖，
// 才能调用第一个invoke方法

```



**重载和重写**



重载：

- 方法名相同  但是参数类型不同

- 在编译时识别出 具体要调用哪一个方法

  三个阶段：

  上面一个阶段找不到 就到下面一个阶段

  同一阶段中找到多个 就选择最适配的（有 父类和子类 就选子类）

  1. 不考虑自动装拆箱和可变长参数
  2. 允许自动装拆箱 不允许可变长参数
  3. 允许自动装拆箱 允许可变长参数

- 也可以重载父类的方法



重写：

子类中方法与父类中方法同名且参数类型相同

如果是静态方法 则子类的方法会隐藏父类的方法

如果不是静态方法 那就是重写



**JVM的静态绑定与动态绑定**

识别方法的Key:

- 类名
- 方法名
- **方法描述符**：参数类型 + 返回类型



静态绑定：解析时能够直接识别目标方法

动态绑定：运行时根据调用者的动态类型来识别目标方法



与调用方法相关的指令：

- `invokestatic`： 调用静态方法 （可以直接识别 静态绑定）
- `invokespecial`：调用私有实例方法 构造器 super.父类实例方法或构造器 接口的默认方法 (可以直接识别 静态绑定)
- `invokevirtual`：调用非私有实例方法
- `invokeinterface`：调用接口方法
- `invokedynamic`：调用动态方法



```java
interface 客户 {
  boolean isVIP();
}

class 商户 {
  public double 折后价格(double 原价, 客户 某客户) {
    return 原价 * 0.8d;
  }
}

class 奸商 extends 商户 {
  @Override
  public double 折后价格(double 原价, 客户 某客户) {
    if (某客户.isVIP()) {                         // invokeinterface      
      return 原价 * 价格歧视();                    // invokestatic
    } else {
      return super.折后价格(原价, 某客户);          // invokespecial
    }
  }
  public static double 价格歧视() {
    // 咱们的杀熟算法太粗暴了，应该将客户城市作为随机数生成器的种子。
    return new Random()                          // invokespecial
           .nextDouble()                         // invokevirtual
           + 0.8d;
  }
}
```



**调用指令的符号引用**

编译器会用符号引用来暂时表示目标方法

符号引用：类名 + 方法名 + 方法描述符

```java
// 在奸商.class的常量池中，#16为接口符号引用，指向接口方法"客户.isVIP()"。而#22为非接口符号引用，指向静态方法"奸商.价格歧视()"。
$ javap -v 奸商.class ...
Constant pool:
...
  #16 = InterfaceMethodref #27.#29        // 客户.isVIP:()Z
...
  #22 = Methodref          #1.#33         // 奸商.价格歧视:()D
...
```

类加载的链接阶段会解析符号引用：（静态绑定）

符号引用 -> 

- 指向方法区中方法的指针（静态绑定）
- 方法表的索引（动态绑定）





对于非接口符号引用 解析步骤如下 假设指向的类为C

1. 在C中查找符合名字及描述符的方法
2. 没找到 则在其父类中继续找 直到Object
3. 在直接或间接实现的接口中找 （方法必须是非私有 非静态的）



对于接口符号引用 假设指向接口为I

1. 在I中查找符合名字及描述符的方法
2. 没有找到 则在Object类的方法中找
3. 在I的父接口中找



**动态绑定**

```java
abstract class Passenger {
  abstract void passThroughImmigration();
  @Override
  public String toString() { ... }
}
class ForeignerPassenger extends Passenger {
   @Override
   void passThroughImmigration() { /* 进外国人通道 */ }
}
class ChinesePassenger extends Passenger {
  @Override
  void passThroughImmigration() { /* 进中国人通道 */ }
  void visitDutyFreeShops() { /* 逛免税店 */ }
}

Passenger passenger = ...
passenger.passThroughImmigration();
```



**虚方法调用**

虚方法：

- `invokevirtual`：非私有实例方法（final方法也是静态绑定）
- `invokeinterface`：接口方法



**方法表**

空间换时间：类加载的链接阶段会为每个类都生成一个张方法表

方法表：

- 是一个数组 数组元素为 指向当前类和其父类的非私有方法的指针

- 约束：

  - 子类方法表中包含父类方法表中的所有方法

  - 子类方法在方法表中的索引值，与它所重写的父类方法的索引值相同

![](https://static001.geekbang.org/resource/image/f1/c3/f1ff9dcb297a458981bd1d189a5b04c3.png)



动态绑定的过程：（解释执行）

1. 访问栈上的调用者
2. 获取调用者的动态类型
3. 获取动态类型对应的方法表
4. 通过索引来查找方法表的方法



即时编译的优化方法：

- 内联缓存
- 方法内联



**内联缓存**

缓存：

遇到某个虚方法时 直接查缓存

动态类型 -> 目标方法（子类的方法表的Cache）



多态：

- 单态：只有一种状态 （只有一个子类）

  虚拟机实现：只用单态内联缓存 即每个虚方法 只缓存一个key-value

- 多态：有限多种状态

- 超多态：超多种状态  会有一个具体的数值

  如果缓存不存在 则put

  如果缓存存在但没有命中 则取消缓存 之后都直接走方法表

```java

// Run with: java -XX:CompileCommand='dontinline,*.passThroughImmigration' Passenger
public abstract class Passenger {
   abstract void passThroughImmigration();
  public static void main(String[] args) {
    Passenger a = new ChinesePassenger();
  Passenger b = new ForeignerPassenger();
    long current = System.currentTimeMillis();
    for (int i = 1; i <= 2_000_000_000; i++) {
      if (i % 100_000_000 == 0) {
        long temp = System.currentTimeMillis();
        System.out.print(i < 1_000_000_000 ? "a\t" : "b\t");
        System.out.println(temp - current);
        current = temp;
      }
      Passenger c = (i < 1_000_000_000) ? a : b;
      c.passThroughImmigration();
  }
  }
}
class ChinesePassenger extends Passenger {
  @Override void passThroughImmigration() {} 
}
class ForeignerPassenger extends Passenger {
  @Override void passThroughImmigration() {}
}
```



### 异常处理

组成：

- 抛出

  - 显式：程序中throw

  - 隐式：JVM

- 捕获



**异常的基本概念**

![](https://static001.geekbang.org/resource/image/47/93/47c8429fc30aec201286b47f3c1a5993.png)

> Error指程序不应捕获的异常 程序触发Error时表示执行状态已经无法恢复 需要终止线程或虚拟机
>
> Exception:程序可能需要捕获并处理的异常



异常实例的创建：需要遍历线程所在栈帧 以生成stack trace



**JVM如何捕获异常**

方法 -> 异常表

异常表表项：代表一个异常处理器

- from: [from, to)监控范围
- to
- target:异常处理器起始位置
- 捕获的异常类型



`try-catch`代码块

1. 抛出异常时JVM会从上至下遍历异常表中所有的表项 如果触发的字节码的索引值在[from, to)范围内 则进一步判断异常类是否匹配

2. 如果匹配 则控制流转到target指向的地方

3. 遍历完了整个异常表还不匹配 则弹出当前方法栈帧 再遍历其调用者的异常表

   最坏情况下会遍历当前线程的Java栈上的所有方法的异常表



`finally`代码块：也会生成异常表的表项 监控整个try和catch块中的内容 

复制finally代码块的内容 分别放在try-catch代码块所有正常或异常执行路径的出口处

![](https://static001.geekbang.org/resource/image/17/06/17e2a3053b06b0a4383884f106e31c06.png)



```java

public class Foo {
  private int tryBlock;
  private int catchBlock;
  private int finallyBlock;
  private int methodExit;

  public void test() {
    try {
      tryBlock = 0;
    } catch (Exception e) {
      catchBlock = 1;
    } finally {
      finallyBlock = 2;
    }
    methodExit = 3;
  }
}


$ javap -c Foo
...
  public void test();
    Code:
       0: aload_0
       1: iconst_0
       2: putfield      #20                 // Field tryBlock:I
       5: goto          30
       8: astore_1
       9: aload_0
      10: iconst_1
      11: putfield      #22                 // Field catchBlock:I
      14: aload_0
      15: iconst_2
      16: putfield      #24                 // Field finallyBlock:I
      19: goto          35
      22: astore_2
      23: aload_0
      24: iconst_2
      25: putfield      #24                 // Field finallyBlock:I
      28: aload_2
      29: athrow
      30: aload_0
      31: iconst_2
      32: putfield      #24                 // Field finallyBlock:I
      35: aload_0
      36: iconst_3
      37: putfield      #26                 // Field methodExit:I
      40: return
    Exception table:
       from    to  target type
           0     5     8   Class java/lang/Exception
           0    14    22   any

  ...

```



问题：`catch`代码块在捕获异常后自身又触发了异常 此时finally向上抛出的异常不是 原异常 而是catch代码块中的异常



解决：

- Suppressed异常：抛出的异常可以附带多个异常信息
- `try-with-resources`：语法糖
  - 字节码自动使用Suppressed异常
  - 自动关闭使用的资源`AutoCloseable`



![image-20220126102146795](/Users/yangsiping/Library/Application Support/typora-user-images/image-20220126102146795.png)



![image-20220126102031143](/Users/yangsiping/Library/Application Support/typora-user-images/image-20220126102031143.png)







![image-20220126112853319](/Users/yangsiping/Library/Application Support/typora-user-images/image-20220126112853319.png)





![image-20220126112924154](/Users/yangsiping/Library/Application Support/typora-user-images/image-20220126112924154.png)
