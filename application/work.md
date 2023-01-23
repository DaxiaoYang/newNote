# work

## 11.21todo:

- [ ] 预留人数开发与设计
  1. 接pulsar
  2. 想数据格式 自己之后查的时候需要什么
  3. 兼容异常
  4. 想一下查询的逻辑 性能 兼容异常
- [ ] 分享ID测试





11.22todo

- [ ] 组队跳游戏数据部分

  

- [ ] 预留人数开发与设计 先弄这个 优先级高 而且卡住别人了

  1. 接pulsar
  2. 想数据格式 自己之后查的时候需要什么
  3. 兼容异常
  4. 想一下查询的逻辑 性能 兼容异常

- [ ] 分享ID测试



```json
<dependency>
<groupId>com.metaverse.cobblestone.room</groupId>
<artifactId>metaverse-room-service-api</artifactId>
<version>1.0.3</version>
</dependency>

com.metaverse.cobblestone.room.api.common.enums.RoomEventMqTopicEnum
```



```java
com.metaverse.runtime.api.facade.game.GameIdTransitionServiceI
```





11.23todo

- [ ] 好友匹配了解一下对面的接口 先把逻辑写上

  问题:

  - MW游戏心跳是指什么 通过什么感知 优势是什么
  - 用户在玩哪个游戏
  - 为什么不需要房间信息 只有游戏信息有什么用 好友房间列表

- [ ] 组队跳游戏

- [ ] 同账号登录



11.24

- [ ] ds memory profile 先给016 017 018分支
- [ ] ds profile功能重复生成自测
- [ ] 好友匹配 人数逻辑补上 补到就剩对方接口的程度



11.27

- [ ] 连接ds埋点讨论 再梳理一下整个连接过程
- [ ] 好友匹配开发完成
- [ ] 了解一下现在的扩缩容逻辑





11.28todo

- [ ] 好友匹配通知editor 检查自身逻辑 pandora埋点注册
- [ ] 埋点的先把我能写的写好 把技术文档写好
- [x] 主动删除cgroup合master 代码合019





11.30todo

- [ ] 好友优先匹配
- [ ] 埋点的先把我能写的写好 把技术文档写好

```
gameId: 433926

uuid: bf7d5d6520f84724bc3a036e3210f48a(先进的游戏房间)

uuid: db373b5d965643f88dacce7fd1a3fa50（后进的游戏房间）

请求
2022-12-02 09:46:53.834  INFO 1 --- [0880-thread-197] c.m.c.room.service.RoomService           : getSocialRoom queryOnlineFriend request:OnlineFriendQuery(uid=db373b5d965643f88dacce7fd1a3fa50, gameId=433926)

返回
2022-12-02 09:46:53.838  INFO 1 --- [0880-thread-197] c.m.c.room.service.RoomService           : getSocialRoom queryOnlineFriend response:ResponseEntity(code=200, message=OK, data=[])
```





12.2todo

- [x] 预留玩家功能调通
- [x] push一下连接ds成功率降低的问题
- [ ] ds打包单独修改方案思考
- [ ] DS在被oom掉的时候 想办法在ds日志中显示出来
- [ ] 想办法抽象一下进入游戏时候的异常报错与错误返回码





12.5todo

- [x] MWTSLIB开发 记得改下时间戳
- [x] cgroup信号了解  在ds被oom kill掉的时候 加上堆栈 内存快照（最好也能放到日志里面） 埋点上报
- [ ] ds入参想方案
- [x] 合代码 部署角色分享服务到pre
- [ ] 改密码

```
developer/login2S:{ "token": "5cf301c34781413b80a2de36570f8847" }

developer/openTsLog2S:{
    "dsId": "P_bcdbf8f2339ce55e0cb6b815823fbc398e91a090-02e672da510644a18fdf0a3431513de9",
    "open": true
}
```



