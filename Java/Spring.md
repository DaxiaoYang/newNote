# Spring

[TOC]

## Spring总览

![image-20210904101502897](/Users/yangsiping/Library/Application Support/typora-user-images/image-20210904101502897.png)

**核心特性**

- IOC容器
- AOP
- Spring事件
- 资源管理
- 国际化
- 校验
- 数据绑定
- 类型转化
- Spring表达式



**数据存储**

- JDBC
- 事务抽象
- DAO支持
- ORM
- XML编列



**编程模型**

- 面向对象编程

  - 契约接口

  - 设计模式

  - 对象继承

- 面向切面编程

  - 动态代理

- 面向元编程
  - 注解
  - 配置
  - 泛型
- 函数驱动
- 模块驱动



**Spring Framework中的核心模块**

- core: 资源管理 泛型处理
- beans: 依赖查找 依赖注入
- aop: 动态代理 
- context: 事件驱动 注解驱动 模块驱动
- expression： 表达式语言



## 重新认识IOC

**IOC定义**

程序的控制流由程序员交到了框架手中，框架会调用我们编写的代码  就是好莱坞原则 *don't call us, we will call you*



**IOC两种主要的实现方式**

- 依赖查找：由容器提供API
- 依赖注入：依赖关系的建立全部交由容器管理
  - 构造器注入
  - setter注入



**IOC容器的职责**

- 依赖处理
  - 依赖查找
  - 依赖注入
- 生命周期管理
  - 容器
  - 托管的资源
- 配置
  - 容器
  - 外部化配置
  - 托管的资源



**依赖查找和依赖注入的区别**

- 依赖查找
  - 主动获取依赖
  - 实现繁琐
  - 侵入业务逻辑
  - 依赖容器API
  - 可读性好
- 依赖注入
  - 被动获取依赖
  - 实现遍历
  - 低侵入性
  - 不依赖容器API
  - 可读性一般



**关于构造器注入和Setter注入**

Spring提倡使用构造器注入 原因如下

- 使组件成为不可变对象 且保证所需的依赖非空
- 当调用组件时 获取的对象都是已经完成初始化的
- 太多的构造参数 也可以提示我们 该类的职责可能不单一 需要重构
- setter注入只应该在非必须的依赖上使用 否则在使用该依赖时需要检查是否非空 setter注入的好处在于可以再次给对象进行注入



**Spring作为IOC容器的优势**

- 典型的IOC管理 依赖查找和依赖注入
- AOP抽象
- 事务抽象
- 事件机制
- SPI扩展
- 第三方整合
- 易测试性
- 更好地面向对象



## Spring IoC概述



**依赖查找**

- 按名称
  - 实时的
  - 延迟的
- 按类型
- 按名称 + 类型
- 按注解



**依赖注入**

- 按名称
- 按类型
- 注入容器内置Bean
- 注入容器内置依赖（不是Bean）



**依赖来源**

- 自定义Bean
- 容器内置Bean
- 容器内置依赖



**配置元信息**

- Bean配置
  - xml
  - properties
  - 注解
- 容器IOC配置
  - xml
  - 注解
- 外部化属性配置 `@Value`
  - 注解



**BeanFactory与ApplicationContext**

- BeanFactory是底层IOC容器
- ApplicationContext是具备应用特性的BeanFactory超集
  - AOP
  - 事件
  - 注解
  - 配置元信息
  - 国际化
  - Environment抽象



**IOC容器的启停**

```java
				// 创建应用上下文
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 将当前类 AnnotationApplicationContextAsIoCContainerDemo 作为配置类（Configuration Class）
        applicationContext.register(AnnotationApplicationContextAsIoCContainerDemo.class);
        // 启动应用上下文
        applicationContext.refresh();
        // 关闭应用上下文
        applicationContext.close();
```



**什么是SpringIoC容器**

通过依赖查找和依赖注入实现了控制反转原则



**BeanFactory和FactoryBean**

- BeanFactory：底层IOC容器
- FactoryBean：创建Bean的一种方式



**Spring容器启动时做了哪些工作**

`refresh()`

- IOC配置元信息读取与解析
- IOC生命周期
- 事件发布
- 国际化



## Spring Bean基础

**BeanDefinition定义**

>   A BeanDefinition describes a bean instance, which has property values,
>  constructor argument values, and further information supplied by
>   concrete implementations.

是Spring Bean的元信息的接口，包括

- Bean的类名
- Bean的行为配置元素：作用域 自动绑定模式 生命周期回调（xml中都有）
- 其他Bean引用
- 配置设置 如Bean properties

![image-20210912093718165](/Users/yangsiping/Library/Application Support/typora-user-images/image-20210912093718165.png)



**BeanDefinition创建**

- BeanDefinitionBuilder

  ```java
  // 1.通过 BeanDefinitionBuilder 构建
  BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(User.class);
  // 通过属性设置
  beanDefinitionBuilder
    .addPropertyValue("id", 1)
    .addPropertyValue("name", "小马哥");
  // 获取 BeanDefinition 实例
  BeanDefinition beanDefinition = beanDefinitionBuilder.getBeanDefinition();
  // BeanDefinition 并非 Bean 终态，可以自定义修改
  ```

- AbstractBeanDefinition



**Bean的注册**

- xml `<bean>`

- 注解 `@Bean` `@Component` `@Import`

- api

  ```java
  DefaultListableBeanFactory beanFactory;
  // 注册将bean名称与bean元信息注册到beanfactory中（放到一个ConcurrentHashMap中）
  this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
  ```

  

**Bean的实例化**

常规：

- 构造器
- Bean的工厂

特殊：

- 通过`ServiceLoaderFactoryBean` 适配SPI  指定`property=serviceloader` 要加载的类
- `applicationContext.getAutowireCapableBeanFactory().createBean(具体类名)`



**Bean的初始化**

- `@PostConstruct`
- `implement InitializingBean`
- 自定义初始化方法 `init-method`

延迟初始化： `@Lazy`



**Bean的销毁**

- `@PreDestroy`
- `implement DisposableBean`
- 自定义方法 `destroy method`





# Spring揭秘



## Spring框架由来

出现目的：简化J2EE的开发，大部分应用在整个生命周期中都不涉及分布式的问题，所以没必要用很重量级的框架如EJB



概述：

![image-20220506063808736](https://gitee.com/yang_siping/static/raw/master/image-20220506063808736.png)

基于POJO进行轻量级开发

模块：

- IC容器：以依赖注入的方式 管理对象之间的依赖关系
- AOP：补足OOP的缺陷
- Spring MVC
- ORM
- 事务



## IoC基本概念

*Ioc*：inversion of control

获取对象的方式发送了变化，之前是需要被注入对象自己创建并初始化自己的依赖对象，现在是通过配置或注解声明依赖关系，由IoC容器注入已经创建并初始化好的依赖对象



依赖注入的方式：声明依赖

- 构造方法：构造完成后可以马上使用，但是会造成方法参数类型过长
- setter：构造后无法马上使用，有方法名描述性好
- 实现注入专门的接口：代码入侵性强



IoC的好处：

- 可以通过依赖注入 注入同一个接口的不同实现类来复用代码 提高灵活性
- 可以依赖注入Mock对象 提高测试性



## IoC Service Provider

> Spring IoC容器算是IoC Service Provider的一个实现



职责：

- 业务对象的创建与初始化
- 业务对象的依赖绑定：保证每个业务对象在使用的时候 都处理就绪状态 而不会报`NullPointerException`



管理对象间依赖的方式：IoC容器怎么知道依赖关系

- 直接编码，手动将对象注册到容器中
- 配置文件
- 注解声明：实际上还是配置 



## BeanFactory

![image-20220510213701447](https://gitee.com/yang_siping/static/raw/master/image-20220510213701447.png)



**Spring中的两种容器类型**

- BeanFactory：

  提供完整的IoC服务支持：对象注册与对象依赖的绑定

  延迟初始化 容器启动速度快 适合所需资源有限的场景

- ApplicationContext

  BeanFactory的一个实现 除了IoC还有其他功能：资源加载 事件发布 国际化支持 

  立即初始化 容器启动速度慢 适合资源充足的场景

  

BeanFactory的作用：pull -> push

原来自己创建对象(pull) 现在由容器将依赖注入(push)



**BeanFactory的对象注册和依赖绑定方式**

- 直接编码方式

  使用`DefaultListableBeanFactory` 

  既是BeanFactory的子类（所以可以访问容器内管理的Bean）

  也是BeanDefinitionRegistry的子类（所以可以注册和管理Bean）

  来将对象的配置元数据(beanDefinition)注册到容器中 并手动指定依赖关系

  然后获取构建好的对象

- 外部配置文件：XML

- 注解方式：注解扫描 将依赖关系和配置信息放到注解上（注解的本质就是类 属性 方法的元信息）



**BeanFactory的XML配置**

<bean> 中的一个属性 `autowire` 自动绑定 无需手动指定该bean定义相关的依赖关系

5种自动绑定方式

- no：不采用自动绑定
- byName：实例变量名与bean的id相同
- byType: 实例类型与bean的类型相同（要求该类型只能有一个实例）
- constructor：通过构造器指定实例类型 本质上跟byType一样
- autodetect：constructor和byType的结合



自动绑定的优缺点：

优点：

- 减少手动配置的工作量
- 某个对象增加依赖关系时 如果当前容器中有相应对象存在 就不需要修改配置

缺点：

- 依赖关系不如写在配置文件中明确
- 某些情况下会出现异常：byType绑定下 同类多实例 byName情况下 之前的id被修改



延迟初始化的bean要保证自身不被不延迟初始化的bean依赖 



bean的scope：bean的存活的限定场景（作用域），bean在进入scope前应该被生成和装配好，不在处于scope时应该被容器销毁

scope类型：

- singleton（默认）：容器内的单例（平时说的单例模式是指某个类加载器下的单例）所有引用共享一个实例 在第一次被请求时初始化 在容器销毁时销毁
- prototype：每次都会生成一个新的实例对象给请求方 容器在返回对象后 不再持有对对象的引用 由请求方管理返回对象后面的生命周期 （一般使用在一些有状态的或不能共享的实例对象上）
- request：作用域为一个HTTP请求
- session：作用域为一个服务器session
- global session：只在portlet的Web中有意义



工厂方法与FactoryBean：

使用场景：

```java
public class Foo {
  	
  private BarInterface barInstance;
  public Foo() {
    // 应该避免这种硬编码 强耦合
    barInstance = new BarInterfaceImpl();
  }
  
}
```

解决方法：

- 我们自己设计的类：通过容器的依赖注入来帮助我们解除接口和实现类之间的耦合性

- 第三方库中的类：工厂方法模式

  具体实现类变更时 被注入对象不需要做任何变动 只变更工厂类就可以

  ```java
  public class Foo {
    	
    private BarInterface barInstance;
    public Foo() {
      // 静态工厂	xml中配置factory-method
      barInstance = BarInterfaceFactory.getIntance();
      // 工厂方法 xml中配置factory-method和factory-bean
      barInstance = new BarInterfaceFactory().getInstance();
    }
  }
  ```



FactoryBean：就是一个bean 但是本身是生产某个对象的工厂

使用场景：

- 某些对象实例化过程过于复杂 相比于XML配置 在Java代码上实现更方便
- 间接注册第三方库中的类到容器中



**容器的实现**

容器作用：加载某种方式存放的配置元信息，根据信息构建对象以及依赖关系，组装成一个可用的轻量级容器的应用系统

两个阶段：

- 容器启动：侧重于对象信息的收集

  通过BeanDefinitionReader读取配置信息为BeanDefinition

  再将其注册到BeanDefinitionRegistry中

- Bean实例化：BeanDefinition -> Bean

  主动或被动调用getBean方法 来实例化对象



容器的启动中提供的扩展点：`BeanFactoryPostProcessor`

可以在实例化Bean之前 对注册到容器的BeanDefinition做修改，会处理容器中所有符合条件的BeanDefinition

3个实现：

- PropertyPlaceholderConfigurer：将XML配置文件中的占位符替换为properties文件中的值（通过修改BeanDefinition）
- PropertyOverrideConfigurer：覆盖XML已经配置的值(通过修改BeanDefinition)
- CustomEditorConfigurer：向容器中注册自定义的PropertyEditor（XML中的字符串 -> Java中的具体类型）



Bean的生命周期

Bean实例化的触发：`BeanFactory.getBean()`

- 显示调用：被客户端显示调用getBean方法
- 隐式调用：
  - 对于BeanFactory来说 Bean默认都是延迟初始化 getBean(A) 会先初始化A所依赖的对象B C等
  - ApplicationContext在启动时会将所有注册到容器中的Bean 都调用一遍getBean

![image-20220516211936067](https://gitee.com/yang_siping/static/raw/master/image-20220516211936067.png)

> 只有Singleton的Bean生命周期是由容器维护的



1. Bean实例化与BeanWrapper：

实例化策略：InstantiationStrategy

- SimpleInstantiationStrategy：反射方式创建实例
- CglibSubclassingInstantiationStrategy：动态字节码生成某个类的子类 可以实现方法注入（默认实现）

此时返回的是一个BeanWrapper，目的是为了方便后续转换类型、设置对象属性（创建时 会将注册的PropertyEditor复制一份给其实现类BeanWrapperImpl）



2. Aware接口

在对象实例化和其依赖设置完成后，依赖注入特定的Bean， xxxAware就注入xxx的实例 



3. BeanPostProcessor

存在于对象实例化阶段，会处理容器内所有符合条件的实例化后的对象实例

使用场景：

- 处理标记接口实现类

  如各种Aware接口的依赖注入 如ApplicationContextAwareProcessor

- 为对象提供代理实现



4. InitializingBean和init-method

自定义补充Bean的初始化逻辑



5. DisposableBean和destroy-method

Bean初始化完成后 容器会检查Bean是否实现了相关销毁方法接口 如果实现了 就注册一个对应的Callback，容器关闭时，在一个shutdownHook里面执行所有的销毁Callback



## ApplicationContext

**统一资源加载策略**

出现背景：

JavaSE提供的java.net.URL只能提供发布在网络上的资源的查找和定位功能，其次没有界别出资源的表示和资源的查找这两个抽象

但是资源本身存在的形式是多样的：

- 序列化后的二进制对象
- 字节流形式
- 文件形式

资源的位置是多样的：

- 文件系统
- Classpath
- URL可以定位的地方



Spring中的抽象：

- Resource：资源的抽象

- ResourceLoader：资源加载策略的抽象

  ResourcePatternResolver：可以借助通配符实现批量查找的ResourceLoader

ApplicationContext与ResourceLoader的关系：

AbstractApplicationContext是ResourcePatternResolver的一个子类



**国际化信息支持**

为不同国家和地区的用户提供对应的语言文件信息

JavaSE中相关的类：

- Locale：代表某个国家和地区
- ResourceBundle：一个组文件 basename + 不同地区的locale名.properties 



Spring中对国际化信息的抽象：MessageSource



**容器内部事件发布**

观察者模式的实现方式之一

JavaSE提供的事件发布：

- 事件：EvenObject
- 事件监听器：EventListener
- 事件发布者：自定义的类
  - 管理事件监听器
  - 发生事件时 触发监听器

![image-20220520161211990](https://gitee.com/yang_siping/static/raw/master/image-20220520161211990.png)



Spring提供的事件发布：

- 事件：ApplicationEvent
- 事件监听器：ApplicationListener
- 事件发布者：ApplicationContext



**基于注解的依赖注入**

- `@Autowired` 

  注解版自动绑定 默认也是byType 根据类型绑定

  `@Qualifier`

  byName自动绑定 注解版

- 用JSR250中的注解来标注bean的依赖关系的和生命周期

  使这些注解生效的类（注解处理器）：CommonAnnotationBeanPostProcessor

  - @Resource byName 不指定名字默认为属性名
  - @PostConstruct
  - @PreDestroy

- classpath-scanning：bean的注册通过扫描包中的注解实现





## 一起来看AOP

OOP与AOP：

- OOP：对业务需求进行抽象与封装

- AOP：是对OOP的一种补充 对系统需求进行模块化的组织

  > 系统需求：日志记录 权限校验 事务管理 
  >
  > 每个接口在请求处理过程中都需要做的事情

![image-20220524210117105](https://gitee.com/yang_siping/static/raw/master/image-20220524210117105.png)



AOP的实现：

- 静态AOP：

  1. 定义Aspect文件
  2. 在字节码编译中将横切关注点织入相应的代码位置中
  3. 将生成的Class文件放入JVM执行

  优点：不会带来性能损失

  缺点：修改需要重新定义Aspect文件 重新编译 加载 运行

- 动态AOP

  在系统运行后 才执行AOP逻辑的织入 可以动态更改织入逻辑

  优点：灵活 易用

  缺点：一般通过操作字节码来完成织入 会带来一定的性能损耗



Java平台上的AOP实现：

- 动态代理

  Spring AOP默认实现 在运行期间 为相应接口动态生成代理对象 

  方法执行通过反射 性能稍微差一点

- 动态字节码增强

  cglib或asm等字节码生成工具 在运行期间 动态生成某个类的子类 

  从而将横切逻辑织入系统

- Java代码生成 静态AOP 比较老

- 自定义类加载器

  加载class文件的时候 织入横切逻辑 

  最后交给JVM运行的是 改动后的class文件

- AOL扩展



AOP中的对象：

- Joinpoint

  织入位置

- Pointcut

  对织入位置的描述

- Advice

  存放织入的横切逻辑 类似于class中的method 

- Aspect

  包含多个pointcut核advice 类似于class

- 织入器

  将AOP中的advice织入到OOP中对应的位置的工具

  Spring AOP中常用的是 ProxyFactory

  

- 目标对象

  被织入逻辑的原始类

![image-20220524214229706](https://gitee.com/yang_siping/static/raw/master/image-20220524214229706.png)





## Spring AOP概述及其实现机制

**概述**：

Spring AOP以Java作为AOP的实现语言(AOL)，默认使用动态代理



**实现机制**

为什么需要动态代理，静态代理有什么问题：

静态代理 对不同的目标对象类型 需要手动生成不同的代理类 并将相同的横切逻辑放入其中

动态代理：

作用：可以为执行的接口 在运行期间 动态地生成代理对象 执行相应的方法都是通过反射

缺点：只能对实现了相应Interface的类使用



Spring AOP默认先尝试动态代理 如果目标对象没有实现任何接口 则尝试使用动态字节码生成cglib(code generation library) 使用继承继续扩展(final类不能用)



## Spring AOP一世

> Spring AOP的各种概念实体的表现形式和使用方式



**Jointpoint**

只支持方法执行，原因：

- KISS 
- 类中属性的Jointpoint会破坏封装的特性，并且对setter getter方法拦截 效果也是一样的
- 需求特殊可以用其他AOP实现 如AspectJ



**Pointcut**

- 确定Advice作用的类范围
- 确定Advice作用的方法范围



**Advice**

- per-class Advice 最常用

  为目标对象的所有实例所共享

- per-instance Advice

  会为不同的目标对象保存各自的状态 Intruduction



**Aspect**

Advisor：里面包含一个Pointcur和一个Advice

- PointcutAdvisor
- IntroductionAdvisor：只能用于类级别 只能用Introduction的Advice

- Ordered：当某个Joinpoint需要执行多个Advice时，如果需要考虑执行顺序 则可以指定执行优先级



**织入：ProxyFactory**

输入：

- 目标对象
- Advisor

两种方式：可以通过配置 告知ProxyFactory使用哪种方式

- 基于接口：目标对象至少有一个接口 调用目标对象的方式是反射调用 性能较差
- 基于类：目标对象没有接口 调用目标对象的方式是调用父类方法 性能较差



ProxyFactory的两个父类/接口：

![image-20220526225722567](https://gitee.com/yang_siping/static/raw/master/image-20220526225722567.png)

- AdvisedSupport：生成一个代理对象的配置信息

  ![image-20220526225500150](https://gitee.com/yang_siping/static/raw/master/image-20220526225500150.png)

- AopProxy：提供生成一个代理对象的功能

  ![image-20220526225446867](https://gitee.com/yang_siping/static/raw/master/image-20220526225446867.png)



## Spring AOP二世

两种新方式：

- @AspectJ加在一个类上

  类中可以包含多个Pointcut和Advice

- 基于Schema

  Java5之前的版本想要使用AspectJ的方法



**@AspectJ形式的Spring AOP**

**织入方式**

- 编程方式：AspectJProxyFactory 
- 自动代理：AnnotationAwareAspectJAutoProxyCreator自动搜集容器中注册的Aspect 并将Advice织入到Pointcut定义的各个目标对象上



- Pointcut: 注解内容最终会转化为AspectJExpressionPointcut

  - 表达式：规定匹配方法的规则

    表达式中的标志符：

    - execution：指定方法签名
    - within：指定类
    - this target：指定代理对象和目标对象
    - args：指定参数
    - @within/@target：指定类上的注解
    - @args：指定参数上的注解
    - @annotation：指定方法上的注解 方法级别

  - 签名：供Aspect中其他地方引用 private类型的只能在本类中引用 public的其他Aspect类也可以引用

- Advice

  可以使用Pointcut表达式 也可以引用Pointcut signature

  必须指定第一个参数为ProceedingJoinPoint类型



**Advice的执行顺序**：当多个Advice匹配到同一个方法的时候

- 同一个Aspect内的Advice 先声明的优先级高 （Before 是优先级高的先执行 AfterReturning是优先级高的最后执行）
- 不同Aspect 可以用Ordered接口指定Aspect类的优先级



<<<<<<< HEAD
**Aspect的实例化模式**

- singleton：容器内单例 默认
- perthis/pertarget：为每个代理对象/目标对象生成一个



**获取当前调用的代理对象**

> 场景：被代理对象method1 中调用method2时 通过代理对象 执行method1 则只会执行method1的代理后的方法 不会执行method2的被代理的方法 
>
> 所以如果需要method2 也执行proxy的方法 就需要获取代理对象的引用

获取方式：

1. 生成代理对象时 设置exposeProxy属性为true 默认为false 因为有性能影响
2. 在被代理对象中使用`AopContext.currentProxy()`获取代理对象

缺点：与Spring AOP框架代码耦合

解耦方法：通过IoC容器注入

- 依赖注入：构造方法/ setter方法
- 方法替换：getThis() 替换为返回proxy对象
- 让被目标对象依赖于一个中间的wrapper类，wrapper类中声明一个getProxy的方法 让目标对象与Spring API解耦
- 定义接口 通过BeanPostProcessor覆盖掉getProxy的方法逻辑





## AOP应用案例

> 即找到系统中的横切关注点 并将其用Aspect的方式定义出来

- 异常处理

  异常类型：

  - checked exception：对应系统中的罕见的非正常状态 提供的信息的面向程序的 让应用程序可以根据不同的异常类型进行针对的处理

  - unchecked exception：对应系统中的严重异常情况 提供的信息是为了让系统维护人员能够判断问题 从而进行人工干预

    对于unchecked exception所做的事情实际很少：记录日志 通知相关人员 所以也是一个横切关注点

- 资源访问控制

- 缓存


## 统一的数据访问异常层次体系

**DAO模式的背景**

面向接口编程，对于不同的数据存储实现，都使用统一的DAO层接口，

隔离了数据访问和存储



问题：不同的数据存储实现，在访问和操作存储的数据时会抛出不同的checked exception，这样会导致客户端使用的接口也必须声明抛出相应的异常，这样dao接口就无法真正起到隔离变化的作用

解决方法：让不同的实现都抛出RuntimeException 并且在实现中解析出异常信息 封装到统一的异常里面 这样客户端就可以通过统一的异常catch来进行相应的处理

轮子：Spring提供的 `org.springframework.dao.DataAccessException`



## JDBC API的最佳实践

JDBC原生API使用中的问题：

每次进行数据操作，都需要执行相同的一系列动作：1 2 5 6都是不变的部署

1. 获取数据库连接 
2. 获取statement
3. 执行sql（变化）
4. 处理结果集（变化）
5. 异常处理
6. 关闭资源



所以可以用模板模式，将不变的部分封装为模板，变的部分抽象为输入的参数（行为参数化，即对已经获取的Connection对象进行的操作）



JdbcTemplate做的事情

- 用模板模式封装JDBC代码

  模板方法 + callback

- 对JDBC的异常转译为Spring的异常



**数据访问中的多数据源**

- “主权独立”的多数据源

  每个数据源都对外独立承担公开数据库资源的职能

  场景：

  - 数据库A存储交易信息 数据库B存储配置信息
  - 读写分离

- “合纵连横”的多数据源

  负载均衡 只不过由一个数据源来负责转发请求

  场景：

  - 高可用
  - 将数据访问类与后面的具体的数据源解耦
  - 负载均衡



## 有关事务的楔子

事务存在的目的：

在软件访问数据源中保存的自身的状态的时候，需要保证系统始终处于“正确”的状态，所以需要对访问行为做一些限制

事务：以可控的方式对数据资源进行访问的一组操作

事务特性：ACID



**事务中的参与者**：

- Resource Manager：负责存储并管理系统数据资源的状态 如数据库
- Transaction Processing Monitor：在分布式事务中协调多个RM的事务处理
- Transaction Manager：TPM中的核心模块，直接负责多个RM之间的事务处理的协调，提供事务界定，事务上下文传播的接口
- Application：事务边界的触发点



**事务类型**：根据涉及的RM数量划分

- 局部事务：1RM 一般用RM内置事务支持

- 分布式事务（全局事务）: 多RM + TPM（两阶段协议：必须所有的RM都同意 才提交事务）

  ![image-20220604095854097](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206040958352.png)



## Java事务管理

- 局部事务支持：

  通过具体的数据访问技术提供的connection的API来管理事务

- 分布式事务支持：

  JTP JCA



存在的问题：

- 局部事务的事务管理依赖于具体数据访问技术(JDBC Hibernate)提供的API，使得事务管理代码与数据访问代码耦合，与业务逻辑代码耦合

- 事务的异常处理

  没有统一的事务相关异常体系，事务处理过程出现的异常一般都是不可恢复的，所以应该抛出unchecked exception而不是checked exception

- 事务管理API过多，没有一个统一的方式来抽象单一的事务管理需求，开发人员应该只关心界定事务的边界

- 想要使用声明式事务，需要依赖EJB容器



解决方法：主要是为了提供开发效率 简化开发过程

- 抽象不同的事务API，将事务管理与数据访问解耦
- 转译checked exception，屏蔽异常差异性
- 对事务的界定操作进行统一的抽象
- 为普通Java对象提供声明式事务



## Spring事务的架构

设计基本原则：将事务管理的关注点与数据访问的关注点相分离

- 在业务层 使用事务的抽象API时进行事务界定时，不需要关心事务资源是什么
- 在数据访问层访问时，不需要关心当前使用的事务资源是否需要参与事务或如何参与事务



核心接口：PlatformTransactionManager

- 获取事务
- 提交事务
- 回滚事务

如何实现：

难点：

因为JDBC局部事务中，要保证两个DAO方法在同一个事务中，就必须保证他们使用同一个java.sql.Connection，而如果通过DAO方法参数显示传递，就会造成数据访问与事务管理耦合

解决方法：通过线程私有数据，将事务资源放在线程私有数据中进行传输，事务提交或者回滚后，将事务资源从线程私有数据中清除

遗留的问题：

- 如何保证相应的接口方法被正确的顺序调用
- 现在需要强制每个数据访问对象通过PlatformTransactionManager获取connection



**Spring事务接口体系**

![image-20220605162148073](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206051621216.png)

- PlatformTransactionManager：界定事务边界
- TransactionDefinition：定义事务相关事务，隔离级别、传播行为等
- TransactionStatus：事务开启到结束期间的事务状态



**TransactionDefinition**：规定了事务的属性

- 隔离级别

- 传播行为

  概念：表明事务中涉及的DAO方法，将以什么样的行为参与事务

  分类：

  - required：

    默认选项，如果存在事务，则加入当前事务，如果不存在任何事务，则创建一个新的事务，至少保证在一个事务中运行

  - requires_new：

    不管是否已经存在事务，都会创建新的事务，当前存在事务时，会将其挂起

    场景：该方法做的操作不想影响到外层事务，比如更新日志信息，就算更新失败，也不想因此影响到外层事务的成功提交

  - supports：

    存在事务，则加入当前事务，不存在事务也直接执行

    场景：适合于查询方法，直接执行查询方法 不需要事务的支持，而如果在其他事务中，加入当前事务能保证其观察到当前事务对数据资源所做的更新

  - not_supported：

    不支持当前事务，方法在没有事务的情况下执行，如果当前存在事务，则将其挂起

  - mandatory：

    强制要求存在一个事务，如果不存在则抛出异常

  - never：

    如果当前存在事务，则抛出异常

  - nested：

    不管是否已经存在事务，都会创建新的事务，当前存在事务时，会在当前事务的一个嵌套事务中执行

    场景：将一个大的事务，划分为多个小的事务来处理，并且外层事务可以根据内层事务的执行结果，来选择不同的执行流

- 超时时间

- 是否为只读：只是提供给RM优化的一个提示



TransactionDefinition相关实现：

分类：

- 编程式事务场景
- 声明式事务场景

![image-20220605232328895](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206052323977.png)



**TransactionStatus**

提供的功能：

- 查询事务状态
- 标记当前事务 使其回滚
- 在当前事务中创建内部嵌套事务

![image-20220605232832731](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206052328812.png)



**PlatformTransactionManager**

概念：

- transaction object：

  承载了当前事务的必要信息，用于提供给TransactionManager

- TransactionSynchronization：

  注册到事务处理过程的回调接口

- TransactionSynchronizationManager：

  事务资源connection绑定到的地方



继承层次：

![image-20220606082257616](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206060822696.png)



AbstractPlatformTransactionManager中固定的处理逻辑：



![image-20220606083316768](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206060833827.png)

- getTransaction：

  判断当前是否存在事务，根据结果以及definition中定义的传播行为采取不同的处理方式

- rollback：

  回滚事务

  触发Synchronization事件

  清理事务资源

- commit：

  提交事务

  触发Synchronization事件

  清理事务资源

  如果事务提交过程中出现异常，也需要回滚事务



## 使用Spring进行事务管理

两种方式：

- 编程式

  - PlatformTransactionManager

  - TransactionTemplate

    对PlatformTransactionManager中事务边界的界定和异常处理做了模板化封装，开发人员只需要提供callback即可

    在callback中使得事务回滚的方式：

    - 抛出unchecked exception
    - 通过TransactionStatus将事务标记为rollbackonly

编程式的不足：事务管理代码与业务逻辑代码相互混杂

- 声明式

  事务管理本身就是一个横切关注点，所以可以放到AOP Advice中，@Transactional，给方法添加元信息，这样处理的时候可以通过反射获取信息，推荐标注在实现类的方法上，因为用cglib构建代理对象的时候，无法读取接口上定义的@Transactional数据



## Spring事务管理之扩展篇

ThreadLocal的应用场景：

- 利用线程私有 来避免数据共享 以实现线程安全
- 在执行流中传递数据 避免通过方法参数形式的传递
- 做成线程私有 可以避免使用线程同步带来的性能损耗
- 作为缓存存放的地方



Strategy模式在开发中的应用：

通过统一的抽象，来向客户端屏蔽其所依赖的具体的行为

作用：

- 将使用算法的客户端代码与算法逻辑解耦，客户端只需要跟策略接口打交道，两者独立变化
- 使用key -> algorithm的映射 来避免过多的if-else，映射可以容器注入，避免硬编码





## 迈向SpringMVC的旅程

Magic Servlet现象：

流程控制逻辑 业务逻辑 数据访问逻辑 视图显示逻辑都放在Servlet里面 

缺点：

- 没有将相应的关注点分离 违背了单一职责原则 逻辑无法重用
- 需要在Servlet中维护视图渲染 当视图变动时需要重新编译（前后端不分离）



将视图逻辑从Servlet抽离出来：JSP

新问题：Magic JSP 所有逻辑现在都放在JSP中

解决：将控制逻辑和业务逻辑从JSP中抽离出来 JSP只负责视图 

JSP Model 2：

![image-20220613214016961](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206132140066.png)

MVC模式：

- 控制器：接收视图（前端）的请求 通知模型进行状态更改 选择合适的视图展示给用户
- 模型：封装了业务逻辑和数据状态
- 视图：转发用户请求到控制器 根虎模型数据的变化渲染数据

![image-20220613214509297](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206132145394.png)



项目中控制器的数目：

- 一个应用中有多个Server作为控制器，一个Servlet对应一个Web请求的处理

  缺点：

  - 需要在web.xml里面不断添加映射 使得文件过于臃肿
  - 各自处理Web请求的处理流程(get post) 这些都是可以提取为模板的固定逻辑

- 一个应用中只有一个Servlet作为集中控制器

  - URL映射关系从由Web容器维护到Servlet自己维护
  - 需要将URL分析 流程控制等Web层通用逻辑能够在不同项目中被复用（用模板模式）



![image-20220613220033517](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206132200591.png)



## Spring MVC初体验

Spring MVC的优势：

- 对控制器中的各种关注点进行了分离：
  - HandlerMapping 处理URL映射到具体的处理控制器
  - LocaleResolver 处理国际化
  - ViewResolver 处理灵活的视图
- 面向接口编程 视图做成了接口
- 可以得到Spring IOC AOP等良好的支持





![image-20220613221334674](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206132213771.png)

DispatcherServlet的处理流程：

1. HandlerMapping（Web请求的处理协调人）

   当请求到达Servlet后 找到对应的HandlerMapping实例

   通过它获取到对应的具体处理类Controller

2. Controller（Web请求的具体处理者）

   处理完之后返回一个ModelAndView实例 实例中包含视图名称和模型数据（现在只剩下模型数据了）

3. ViewResolver和View 接口 用于抽象视图的生成策略





# Spring源码



## Spring容器初始化



**简单容器的使用**

![image-20220616215021939](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202206162150080.png)

XmlBeanFactory工作原理：

1. 用ClassPathResource封装xml配置文件
2. 解析获取Bean的全类名
3. 反射创建Bean
4. 将创建好的Bean放入容器中
5. 通过getBean方法获取Bean



**resource**:

对不同资源的抽象，其父接口InputStreamSource，定义了获取输入流的方式，屏蔽了获取不同资源的细节



**忽略指定接口自动装配**

初始化容器的时候，忽略了BeanNameAware BeanFactoryAware BeanClassLoaderAware3个接口的自动装配

ignoreDependencyInterface作用：

将3个接口从依赖检查中排除掉，这3个的依赖注入由BeanPostProcessor进行，而不是用户配置的值，属性的赋值权利交给Spring



**XML文件的解析与校验**

- 解析：DOM解析
- 校验：根据不同格式 DTD XSD分别校验 从Spring库加载自带的校验文件来校验



**Bean标签解析**

解析完后创建BeanDefinition对象，BeanDefinition存储用于实例化Bean的信息

![image-20220803212339614](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202208032123677.png)



**BeanDefinition注册到容器**

将Beanname -> BeanDefinition这对映射放到DefaultListableBeanFactory中的beanDefinitionMap中



**ApplicationContext初始化**

refresh方法

- 准备BeanFactory
- 
