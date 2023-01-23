# Unreal Engine 

[TOC]

  

## 网络复制

> 概念：将服务端的Actor对象信息同步客户端上

 

网络模式：一个游戏实例在游戏会话中扮演的角色

- Standalone：

  游戏实例(Game.exe 一个玩家正在玩的游戏进程)既是客户端也是服务端 但是不对其他客户端开放 如单机单人游戏 或单机多人游戏

- DedicatedServer： 

  没有本地玩家也没有视口（做渲染工作） 只用于作为服务端 供客户端连接

- ListenServer：（所以相当于是在客户端起的Server）

  游戏实例既是客户端也是服务端 可以对其他客户端开放

- Client



![image-20220513112550971](https://gitee.com/yang_siping/static/raw/master/image-20220513112550971.png)

层级：

> GameServer.ext/GameClient.ext
>
> > UGameEngine
> >
> > > UNetDriver
> > >
> > > > UNetConnection：相当于client - server之间的socket 与一个PlayerController对象
> > > >
> > > > > UActorChannel ：用于同步Actor对象（Actor对象是网络复制的基本单位）

 

当一个Actor对象从服务端被复制到客户端后可以做的事情：

- 生命周期同步：客户端的Actor对象的生命周期跟随服务端的生命周期
- 属性同步：服务端Actor对象属性修改可以同步到客户端
- RPC：可以通过调用Actor对象上的函数实现RPC（多播 单播(c -> s / s -> c)）



Actor对象的所有权：

UNetConnection -owns-> PlayerConntroller -> Actor对象 



Actor的相关性：决定Actor对象什么时候需要复制 以及需要复制时 发送到哪些NetConnection

影响因素

- 所有权
- Actor对象与玩家的空间距离



Actor复制方式：

- 周期性同步：主要用于同步属性
  - 可以设置一秒钟检查多少次Actor对象 如果有更新就发送
  - 优先级 宽带有限 优先级高的Actor会优先发送
- RPC：实时性较高



双端动态

静态



 Actor的角色：

要解决的问题：当前运行的代码对于要处理的Actor对象是否有Authority，

有Authority 就表明可以更改这个Actor对象的状态，没有Authority最多只能控制这个Actor的移动和行为 



## 游戏框架

游戏循环：

```C++
int main()
{
  // 初始化
  init();
  // 每一帧：根据输入 更新状态 渲染
  while (!g_exit_requested) 
  {
    poll_input();
    update();
    render();
  }
  // 清理工作
  shutdown();
}
```



UE4中开发：不需要写游戏循环

- 继承GameMode 覆写InitGame方法
- 继承Actor或Component 覆写BeginPlay或Tick方法



UE4初始化时的调用栈

![image-20220516143803243](https://gitee.com/yang_siping/static/raw/master/image-20220516143803243.png)



`Launch.cpp`

![image-20220516113007365](https://gitee.com/yang_siping/static/raw/master/image-20220516113007365.png)

 

**PreInit**

加载所需的引擎 项目和插件模块



**Init**

创建并初始化和Start Engine类 

同时通知Engine生命周期事件

Engine类主要负责的事情：

- Browse：加载某个URL（客户端连接的某个地址） 或游戏地图

  加载地图完成前会初始化3个重要对象：

  - GameInstace：与项目有关 的一些功能
  - GameViewportClient：屏幕
  - LocalPlayer：玩家

- LoadMap：  加载与某个地图相关的信息 （LoadMap之后游戏就进入了可玩状态 就可以进入游戏循环了）

  - 从umap文件中反序列化出一个World对象 World下面是Level 

    Level下面是Actor

  - 对World内的Actor进行几次循环

    - 注册World内的所有Actor
    - 让Actor所属的关卡来初始化Actor

  - BeginPlay：

    调用所有Actor 组件 蓝图的BeginPlay方法

    Engine -> World -> GameMode -> Actor

  

 Actor对象：

- 管理游戏状态的：（游戏框架Actor）
  - GameModeBase：定义游戏规则
  - GameSession：处理登录请求 作为网络服务的接口
  - GameNetworkManager：作弊检测 
  - GameStateBase：服务端生成 保存游戏状态 会被复制到客户端
- 代表玩家的：
  - PlayerController：远程登录的玩家 登录之后生成
  - LocalPlayer：本地登录的玩家
  - Pawn：PlayController所有的对象 用来控制角色（角色死亡 Pawn会被销毁 重生时重新分配一个Pawn给PlayerController）
  - Character：Pawn的子类





