# Effective C++

[TOC]

## 自己习惯C++

### 把C++看出一个语言联邦

C++中的次语言：

- C
- 面向对象
- 泛型编程(模板)
- STL（模板程序库）



### const enum inline 替换 #define

> 本质上是以编译器替代预处理器

- 常量

  - 用#define定义常量 

    当发现错误时 不知道变量名 只知道变量值 难以定位问题

  - 用#define 定义宏

    可以避免函数调用 但是会出各种问题

    推荐使用template inline替代宏 （函数行为可预料 类型安全 执行效率也得到保证 也可以加上类的访问控制）

    


### 尽可能使用const






