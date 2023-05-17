# 深入剖析Kubernetes

[TOC]

```shell
# mac本机启动k8s环境命令
minikube start --driver=docker
```



## 课前必读



**PaaS项目**

- 提供了应用托管的能力 租一台虚拟机 部署Cloud Foundry项目 然后在机器上一键部署应用

  ```c
  cf push "app"
  ```

- 核心功能：应用的打包与分发

  1. 把应用的可执行文件和启动脚本打进一个压缩包里面 上传到云上Cloud Foundry的存储 Cloud Foundry会通过调度器 选择一个可以运行
  2. 这个应用的虚拟机 然后通知这个机器的Agent将应用压缩包下载下来启动
  3. 使用Cgroups和Namespace机制 为每个应用创建一个隔离环境

存在的问题：需要为各个语言 框架 应用版本都打一次包，本地和远程环境不一致



Docker解决了应用打包的问题：

压缩包里面是一个完整的操作系统的文件和目录结构，本地与云端的环境高度一致

打包是执行操作环境+执行文件



**容器编排**：对Docker容器的定义 配置和创建动作的管理（k8s做的事）



## 容器技术概念入门

### 从进程说开去

> 容器 到底是怎么一回事
>
> 容器本质上就是一种特殊的进程
>
> 就是在创建容器进程的时候 指定的这个进程所需的一组Namespace参数 这样容器就只能看到Namespace限定内的文件

容器技术的核心功能：

通过约束和修改进程的动态表现 为其创造一个边界

- Cgroups:制造约束
- Namespace：修改进程视图



```shell
# 启动一个容器 容器里执行/bin/sh 并且分配一个命令行终端跟这个容器交互
$ docker run -it busybox /bin/sh


/ # ps 启动后 Namesapce机制
PID  USER   TIME COMMAND
  1 root   0:00 /bin/sh
  10 root   0:00 ps
  
# 底层：创建新进程时的一个参数设置CLONE_NEWPID 新创建的进程会在单独的一个namespace里面 会认为自己是1号进程
# 1号进程fork出来的子进程 会继承父进程的各种命名空间
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL);   
```



### 隔离与限制

**隔离**

容器相比于虚拟化技术（需要运行一个完整的Guest OS才能执行用户的应用进程 ） 不占用额外资源且直接跟宿主机沟通 没有性能损耗

问题：隔离不彻底

- 多个容器之间还是会共享同一个宿主机的操作系统内核
- 内核中的很多对象没有办法被namespace区分 如系统时间



**限制**：解决部分隔离不彻底的问题

容器使用的资源 如果不加设置 是需要和其他进程一起争用CPU和内存的

所以需要Linux Cgroups来限制一个进程组能够使用的资源上限：CPU 内存 磁盘 网络带宽



### 深入理解容器镜像

问题：容器进程看到的文件系统还是跟宿主机是一样的

解决方法：

- 在容器进程启动前重新挂载它的根目录/ （保证了本地环境与远端环境的一致性）
- 由于Mount Namespace的存在 这个挂载对宿主机不可见 



Docker的核心原理：

- 启用Linux Namespace配置
- 设置Cgroups
- 切换进程的根目录



镜像的制作：

分层 + 目录联合挂载功能

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302142022166.png)





### 重新认识Docker容器

目的：用Docker部署一个Web应用

```py

from flask import Flask
import socket
import os

app = Flask(__name__)

@app.route('/')
def hello():
    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>"           
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname())
    
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

```shell
# 应用依赖
$ cat requirements.txt
Flask
```



**制作容器镜像**

```dockerfile
# 每个原语执行后都会生成对应的镜像层
# 使用官方提供的Python开发镜像作为基础镜像
FROM python:2.7-slim

# 将工作目录切换为/app
WORKDIR /app

# 将当前目录Dockerfile所在目录下的所有内容复制到/app下
ADD . /app

# 使用pip命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的80端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个Python应用的启动命令
CMD ["python", "app.py"]
```



```shell
# 加载dockerfile并执行内容 
$ docker build -t helloworld .

# 生成了一个镜像
$ docker image ls
REPOSITORY            TAG                 IMAGE ID
helloworld         latest              653287cdf998

# 启动容器 容器内的80映射到宿主机的4000端口
$ docker run -p 4000:80 helloworld


$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED
4ddf4638572d        helloworld       "python app.py"     10 seconds ago


$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 4ddf4638572d<br/>
```



```sh
$ docker exec -it 4ddf4638572d /bin/sh
# 在容器内部新建了一个文件
root@4ddf4638572d:/app# touch test.txt
root@4ddf4638572d:/app# exit

#将这个新建的文件提交到镜像中保存
$ docker commit 4ddf4638572d geektime/helloworld:v2
```

**docker exec原理**

一个进程可以选择加入到某个进程的Namespace中，从而进入入这个进程的容器



**docker commit：是在宿主机下执行**

容器运行起来后 将最上层的可读写层 加上原先容器镜像的只读层 打包成了一个新的镜像



**Volume机制**：解决怎么让宿主机和容器的文件互通的问题

```sh
# 将宿主机的home目录挂载到容器的test目录下
$ docker run -v /home:/test ...
```

在 rootfs 准备好之后，在执行 chroot 之前，把 Volume 指定的宿主机目录（比如 /home 目录），挂载到指定的容器目录（比如 /test 目录）在宿主机上对应的目录（即 /var/lib/docker/aufs/mnt/[可读写层 ID]/test）上，这个 Volume 的挂载工作就完成了



```sh
mount -bind /home /test #修改/test的dentry(文件指针) 指向了/home的inode
```

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302170756501.png)



### Kubernetes的本质

k8s要解决的问题：

将应用的容器镜像 放到指定的集群上运行起来 以及提供路由网关 水平扩展 监控 备份 灾难恢复等运维能力

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302170834895.png)



Master节点：控制节点

- api-server：API服务
- scheduler：调度
- controller-manager：容器编排

Etcd：存储持久化数据



Node结点：计算节点

- kubelet：

  - 通过CRI 与docker项目交互  docker项目再通过IOC与linux os交互
  - 通过gPRC与device plugin交互 管理GPU等宿主机物理设备
  - 通过CNI 调用网络插件
  - 通过CSI 调用持久化存储

  

**最主要的设计思想**

从更宏观的角度 以统一的方式定义任务之间的各种关系

声明式API

1. 通过一个编排对象 Pod Job CronJob 来描述应用
2. 为其定义服务对象 负责具体的平台级的功能



- Pod：pod内的容器 共享一个Network namespace 同一组数据卷 从而高效交换信息（需要频繁交互 或通过本地文件交互的应用会放在同一个pod中）
- Service：作为pod的代理入口 对外暴露一个固定的网络地址
- Deployment：一次启动实例的多个应用

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302170856551.png)



**编排**：按照用户意愿和系统规则 自动化处理好容器之间的各种关系 





## k8s集群搭建与实践

### kubeadm：一键部署k8s

```shell
# 创建一个Master节点
$ kubeadm init

# 将一个Node节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口>
```



**kubeadm的工作原理**

让kubelet直接运行在宿主机上 使用容器部署其他的k8s组件



kubeadm init做的事情

1. 检查机器是否可以用来部署k8s

2. 生成k8s所需的证书和目录
3. 为其他组件生成访问kube-apiserver所需的配置文件
4. 为master的3个组件生成Pod配置文件 static pod
5. 同样以static pod方式启动Etcd
6. 生成一个bootstrap token 供后面的join
7. 将master节点信息保存在Etcd中
8. 安装默认插件:kube-proxy DNS 提供整个集群的服务发现和DNS功能



kubeadm join的工作流程

用bootstrap token换取安全验证的角色



### 第一个容器化应用

制作好容器镜像后，需要编写配置文件

```shell
# 运行容器
$ kubectl create -f 我的配置文件
```



```yaml
# 对应一个API对象 k8s会负责创建这个对象 Metadata+Spec
apiVersion: apps/v1
# 类型 定义多副本对象
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  # spec.template Pod模板
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```



**Events**

对API对象的所有重要操作的日志





## 容器编排与k8s作业管理

### 为什么我们需要Pod

> 容器是进程 k8s是操作系统



场景：具有紧密协作关系的一组容器 需要被放到同一个Node节点上

传统解决方法：

- 资源囤积 等到有足够的资源满足一组容器的时 才进行分配
- 不管冲突 后续回滚



k8s解决思路：调度统一按照Pod的资源需求进行计算

会放到同一个Pod中的容器：

- 会发生直接的文件交换
- 使用localhost或socket文件进行本地通信
- 发生频繁的远程调用
- 需要共享Linux namespace



**定义**

Pod 就是一组共享了某些资源的容器

是k8s中的最小编排单位

（Pod与容器类似于进程与线程），所有容器共享同一个Network Namespace，Volume



**容器设计模式**：

> 功能不相关的应用 应该分别放在不同的容器中跑 

siderchar :在一个Pod中启动一个辅助容器 来完成一些独立与主容器之外的操作

tomcat与war包放在不同的容器中 通过volume共享文件





### 深入解析Pod对象（一） 基本概念

Pod级别：调度 网络 存储 安全

字段：

- NodeSelector：将Pod与Node绑定
- NodeName：被调度到的Node节点名字
- Container：容器
  - Imgae：镜像
  - command：启动命令
  - workingDir：工作目录
  - ports：开放端口
  - volumeMounts：要挂载的volume
  - imagePullPolicy：镜像拉取策略 决定每次启动都拉取新镜像 还是只拉取一次
  - lifecyle：生命周期函数



**Pod对象在k8s中的生命周期**

pod.status.phase字段

- pending: api对象已创建并被保存在Etcd中 但是Pod中的有些容器不能被顺利创建
- running:pod已经调度成功 并与某个node绑定 所有容器都创建成功 并且至少有一个在运行
- succeeded: 所有容器都正常结束运行 
- failed:至少有一个容器以不正常的状态退出
- unknown: pod的状态不能持续被kubelet回报给kube-apiserver 可能是通信出了问题





### 深入解析Pod对象（二） 使用进阶

**Projected Volume**：为容器提供预定义好的数据，同时还可以自动更新

- Secret

  把Pod想要访问的数据 放到etcd中

  可以通过容器里挂载volume的方式 访问到这些Secret中的信息 文件中的内容 也会随着etcd中的更新而更新

- ConfigMap：配置文件的对象

- Downward API：将Pod API对象的信息 放到容器里的文件

- ServiceAccountToken：API Server访问的token



**容器健康检查和恢复机制**

- 健康检查探针：自己设置探针的uri (livenessProbe)

- 恢复机制 *restartPolicy*

  默认值为always 任何时候容器异常 都会被重新创建 都是继续在当前Node上

  - always：容器不在运行状态 就重启
  - onFailure：异常时才重启
  - never：从来不重启

  

**Pod状态设计原理**

- 只要策略允许重启异常容器 这个Pod就会一直保持Runnning状态 否则Pod就会进入Failed
- 仅当Pod中的所有容器都进入异常时 才是Failed



**自动填充Pod字段**：PodPreset

PodPreset中定义的内容 会被Pod API对象创建之前追加到这个对象上



### 编排：控制器模型

```go
for {
  // 来自k8s集群本身
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  // 来自用户提交的yaml文件
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    // 创建 更新 删除 API对象
    执行编排动作，将实际状态调整为期望状态
  }
}
```



![img](https://static001.geekbang.org/resource/image/72/26/72cc68d82237071898a1d149c8354b26.png?wh=1920*1080)



### 作业副本与水平扩展

**Deployment对象**：

实现了Pod的水平扩展与收缩，实际依赖的是ReplicaSet对象 不是Pod对象

```yaml

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```



![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302241945090.jpg)

水平扩展/收缩：Deployment Controller只需要修改ReplicaSet中Pod副本个数就行

更新方式：滚动更新

新 ReplicaSet 管理的 Pod 副本数，从 0 个变成 1 个，再变成 2 个，最后变成 3 个。

旧的 ReplicaSet 管理的 Pod 副本数则从 3 个变成 2 个，再变成 1 个，最后变成 0 个

好处：如果新版本Pod无法启动 滚动更新会停止 不会影响到旧版本



应用版本实际上是与ReplicaSet对象一一对应的

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202302241950746.jpg)



### 深入理解StatefulSet(一) ：拓扑状态

**Deployment不足**：

> 可以满足无状态应用 但是不能满足有状态应用的编排

- 不能满足Pod之间有依赖关系的情况（主从 主备）
- 不能满足Pod依赖外部数据的情况（数据存储应用）



**StatefulSet设计**

- 拓扑状态，Pod启动有顺序 

  使用Pod模板创建Pod的时候 对它们进行编号 调谐的时候会严格按照编号顺序进行操作

- 存储状态，Pod的重建 不影响数据的存取

以某种方式记录下有状态应用的状态 并在Pod被重新创建时 恢复这些状态

还是约定大于配置，给有状态的资源（网络标识 文件存储），约定一个根据Pod编号起的固定的名字 Pod重启后 由于Pod名字不变 还是能找到对应的资源



Service的访问方式：

一个Deployment有3个Pod，可以给某个Pod定义一个Service暴露访问

- Service virtual IP：直接访问一个IP地址 会把请求转发到Service代理的Pod上

- Service DNS：访问 `<pod-name>.<svc-name>.<namespace>.svc.cluster.local` 这条DNS 就可以访问到名字为my-svc的Service代理的Pod

  实现：

  - DNS解析到的还是虚拟IP
  - headless serivce：解析到的是Pod的IP地址



```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  # 表明Service创建之后 不会分配一个VIP 而是以DNS记录的方式暴露Pod IP
  clusterIP: None
  selector:
  	# 所有携带app=nginx标签的Pod都会被这个Service代理 是可以一对多的
    app: nginx
```



```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
	# 多了一个服务名
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```



### 深入理解StatefulSet（二）：存储状态

引入Persistent Volume Claim（相当于接口）与Persistent Volume（相当于实现）的API对象 

**使用Volume步骤**

1. 定义一个PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
# 声明想要的Volume属性  
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

2. 在Pod中使用这个PVC

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim
```

 k8s会自动为其绑定一个符合条件的Volume 符合条件的volume来自运维人员维护的PV对象，Pod重启后与原先PVC绑定的PV 还是会绑定到该PVC上（Pod被删除时 PVC与PV都还在）

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
    # 使用 kubectl get pods -n rook-ceph 查看 rook-ceph-mon- 开头的 POD IP 即可得下面的列表
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
```



### 容器化守护进程的意义：DaemonSet

DaemonSet：运行一个DaemonPod 

Pod特征：

- 运行在每一个Node上
- 一个Node只有一个这种pod实例
- 与Node生命周期相同



作用：Node范围的

- 网络
- 存储
- 监控和日志



特点：开始运行的时机 比整个k8s集群出现的时间都早（如何实现 ？ 此时所有Worker节点的状态都是NetworkReady=false）

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```





问题：

**如何保证一个Node上只有一个这种Pod**

1. DaemonSet Controller从Etcd中获取所有的Node列表
2. 遍历所有Node 对Pod节点数量进行修改（调谐）



**如何在指定Node上创建新Pod**

在创建Pod的时候添加字段：

- nodeAffinity：调度相关
- tolerations：可以容忍node的某些污点



### 离线业务：Job与CronJob

作业类型：计算完成之后直接退出

**Job**

```yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never # 永远不应该被重启
  backoffLimit: 4 # 失败重试次数
```

- spec.activeDeadlineSeconds：最长运行时间
- spec.parallelism：并行pod数量
- spec.completions：最小完成数



**CronJob**

定时任务 本质上是Job的一个控制器（controller）

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```





### 声明式API与kubernetes编程范式

**声明式API**

kubectl apply 执行了一个对原有API对象的PATCH操作，kube-apiserver在响应声明式请求的时候，一次能处理多个写操作 具备Merge能力

- 声明式：提交一个定义好的API对象 说明期望的状态

- 允许有多个API写端 以PATCH的方式对API对象进行修改

  



**Istio**

基于k8s的微服务治理框架

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303031216033.jpg)

Dynamic Admission Control：Initializer

在Pod YAML被提交给k8s后 在对应的API对象中自动加上Envoy容器的配置

做法：

1. Envoy容器的定义以ConfigMap的方式保存在k8s中

2. 将编写好的initializer 作为一个Pod部署在k8s中

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     labels:
       app: envoy-initializer
     name: envoy-initializer
   spec:
     containers:
       - name: envoy-initializer
         image: envoy-initializer:0.0.1
         imagePullPolicy: Always
   ```

3. Initializer控制器：往Pod里合并ConfigMap里面的容器字段（容器的PATCH请求） 



### 深入解析声明式API（一）：API对象的奥秘

> 声明式API的工作原理 如何添加自定义的API对象（类似于声明一个Class）

API对象在Etcd的资源路径组成

- Group：API组
- Version：API版本
- Resource：API资源类型

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303061928218.png)



**创建API对象的过程**

1. 向APIServer发起创建API对象的POST请求
1. filters
1. 映射到对应的handler 找到对应的API对象的类型定义
1. 根据类型定义 使用yaml文件中的字段 创建一个API对象 
1. 对API对象的数据进行修改和校验 保存到APIServer的Registry中
1. APIServer把API对象序列化并保存到Etcd中



**如何添加自定义的API类型**

1. 编写CRD custom resource definition的yaml文件

   ```yaml
   apiVersion: apiextensions.k8s.io/v1beta1
   kind: CustomResourceDefinition
   metadata:
     name: networks.samplecrd.k8s.io
   spec:
     group: samplecrd.k8s.io
     version: v1
     names:
       kind: Network
       plural: networks
     scope: Namespaced
   ```

2. 认识具体的字段 添加go代码

3. 让客户端知道这个自定义类型 添加go代码





### 深入解析声明式API（二）：编写自定义控制器

1. 编写main函数

   定义 初始化 启动一个自定义控制器

2. 编写自定义控制器的定义

3. 编写控制器里的业务逻辑



**自定义控制器的工作原理** （这个挺重要的）

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303062021057.png)

1. Reflector从API Server获取关心的类型的事件通知 并放入到队列中

2. 从队列中取出事件 更新本地缓存 并将事件对应的API对象放入到工作队列中

3. 控制循环中 等待本地缓存数据同步完成 然后启动协程 执行对应的业务逻辑 

   业务逻辑中通过API对象的key：namespace/name 到inform的本地缓存中获取API对象（即期望状态）

   然后获取k8s对应API对象的实际状态 有差异则进行调谐(reconcile) 调谐逻辑是自己代码写的



### 基于角色的权限控制：RBAC

*role-based access control*

- role：角色 一组对API对象的操作权限
- subject：被作用者 一般是ServiceAccount
- roleBinding：被作用者与角色的映射



### Operator工作原理

对有状态应用的管理

使用k8s的自定义API资源 CRD 来描述我们想要部署的有状态应用 然后在自定义控制器中根据自定义API对象的变化 来完成具体的部署和运维工作





## k8s容器持久化存储

### PV PVC StorageClass



**PV**: persistent volumn

描述的是持久化存储数据卷 这个API对象定义了一个持久化存储在宿主机上的目录（可以是远端文件系统的挂载目录）



**PVC**： 

描述的是 Pod所希望使用的持久化存储的属性



绑定PVC与PV时需要检查的条件：

- PV spec字段是否满足PVC
- 两者storageClassName字段必须一样



**PersistentVolumeController**

控制循环中 检查每一个PVC 如果有没绑定PV的 从所有可用的PV中绑定



**PV对象如何变成容器里的一个持久化存储**

容器中的Volume：容器中的目录mount了宿主机的目录

持久化Volume：宿主机的目录中的内容 不会随着容器和宿主机的删除而删除 所有其实现往往依赖于一个远程存储服务（NFS等）



准备持久化宿主机目录的过程：两阶段处理

1. attach

   将远程磁盘服务的存储设备挂载到Pod所在的宿主机上

2. mount

   将磁盘设备格式化 并挂载到Volume宿主机目录



问题：PV如何创建

**StorageClass**

通过模板创建 Dynamic Provisioning





### PV PVC的作用：本地持久化卷为例

需求：直接使用宿主机上的本地磁盘目录 不依赖远程存储服务来提供持久化的容器Volume（本地SSD磁盘读写性能比较好）



**Local Persistent Volume**

适用场景：

- 对IO敏感的数据库类型应用
- 具有数据备份和恢复功能的应用 因为节点挂掉且不能恢复时 volume数据就丢失了



难点：

- 本地磁盘如何抽象成PV

  一个PV 对应一快挂载盘

- 如何保证Pod始终被调度到local persistent volume所在的node 

  在调度的时候 考虑volume分布 

  pv nodeAffinity 中指定具体的 node
  
  延迟绑定PVC与PV





## k8s容器网络

### 容器网络

linux容器能看到的网络栈：自己的Network Namespace

- 网卡
- 回环设备
- 路由表
- iptables规则



问题：一个宿主机上的容器之间如何互相交互

利用：docker0网桥（二层设备 通过MAC地址发送）

Veth Pair：一端在容器NetworkNamespace中 另一端在宿主机namespace中 

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303142246002.png)





### 容器跨主机网络

Flannel实现:

- VXLAN
- host-gw
- UDP



etcd：子网 -> node节点的映射



UDP实现：

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303151954259.jpg)

性能问题：

flannel0 会频繁设计内核态与用户态之间的数据拷贝



优化原则：

减少用户态到内核态的切换次数 把核心的处理逻辑都放在内核态进行



VXLAN实现：

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303152018044.jpg)



节点中有的映射：

- 路由映射：子网 -> flannel.1设备
- 目的容器IP -> 目的flannel.1设备MAC

转发数据库中有的映射：

目的flannel.1MAC -> 目的宿主机IP



封包过程：

1. 原始IP包 从容器来到docker0网桥
2. 被路由到flanne.1设备
3. 通过node上的路由信息 获取目的宿主机VTEP设备的IP地址 获取目的宿主机VTEP设备的MAC地址
4. 从转发数据库中 通过目的设备MAC地址 找到目的主机IP
5. 通过ARP找到目的主机MAC地址 封装成正常的二层数据帧（因为都是在同一个子网中 不需要三层设备）

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303152031414.jpg)



解包过程：

1. 目的主机收到数据帧后 发现有VXLAN Header 通过VTEP MAC交给本机的flannel.1设备
2. flannel.1设备取出原始包 根据目的IP地址 通过docker0网桥 发送到目的容器中





### k8s网络模型与CNI网络插件

不同：网桥名字：cni 而不是docker0

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303201944038.jpg)



CNI设计思想：

在k8s启动infra容器后 就可以直接调用CNI网络插件 设置这个infra容器的Network Namespace





### k8s的三层网络方案

**Flannel的host-gw模式**

将每个node上的flannel子网 设置为下一跳地址

将目的宿主机当作跨主容器通信的网关

到达目的宿主机后 根据路由规则 进入到cni0网桥

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303202001720.png)

缺点：要求宿主机之间是二层连通的 即在同一个子网内（公有云下一般是连通的 私人环境下不一定）



**Calico**

也是添加目的容器IP地址段的下一跳的网关地址到源宿主机中

使用BGP来自动分发路由信息，边界网关中拥有其他自治系统里的主机路由信息

跨子网的通信：IPIP驱动 在原IP包外 再包一层IP头

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303202106552.jpg)





### 为什么k8s只有soft multi-tenancy

需要解决的问题：k8s中怎么设计容器的网络隔离

**NetworkPolicy**

```yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  # 入口白名单
  ingress:
  - from:
    # 或的关系
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  # 出口白名单    
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```



原理：NetworkPolicy在控制循环中根据API对象的变动 修改宿主机的iptables规则

> iptables实际上是修改linux内核中的Netfilter子系统
>
> ![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303202152748.png)
>
> 3个检查点
>
> ![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303202152921.jpg)





### 定位到容器 Service DNS与服务发现

> 解决的问题：如何找到某一个容器
>
> 服务发现：如何在Pod（服务）的IP不固定的情况下 如何通过固定的方式访问到这个Pod

```yaml
# service 用于实现负载均衡 暴露一个IP
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
    
# service代理的pod 成为Endpoints  用的是VIP 实际上PING不同 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP    
```



实现：kube-proxy + iptables

当service api对象被提交给k8s kube-proxy就可以通过Service的Informer感知到 会在宿主机上添加一条iptables规则 让进入目的IP和目的端口的IP包 都跳转到另一条iptables链进行处理

 

```shell
# 按概率匹配 到对应的代理的pod 作用修改目的IP和端口 

-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
```

缺点：iptables规则过多 占用宿主机CPU资源

改进：基于IPVS的Service

kube-proxy会在虚拟机上创建一个虚拟网卡 设置Service VIP作为其IP地址 就会通过Linux和IPVS模块 为这个IP设置3个IPVS虚拟主机（基于Netfilter的NAT模式）



也可以通过某个DNS记录 解析到Service的VIP





### 从外界连通Service

问题：Service的访问入口 只能在k8s集群中路由

目的：从k8s集群外 访问到k8s中创建的Service



**NodePort:**通过宿主机的端口 访问Service代理的Pod的端口

实际上就是在宿主机上加了一条 iptables规则 

在POSTROUTING检查点 做了SNAT操作：将IP包源地址 换成了宿主机地址 或者CNI网桥地址

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```



**指定一个LoadBalancer类型的Service**

会调用一CloudProvider在云上创建一个负载均衡服务 并把代理的Pod的IP地址 配置给负载均衡服务做后端

```yaml
kind: Service
apiVersion: v1
metadata:
  name: example-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example
  type: LoadBalancer
```



**ExternalName**

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  # 外部使用 可以映射到k8s DNS地址
  externalName: my.database.example.com
  
  
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  # 或者直接配置外网IP  
  externalIPs:
  - 80.11.12.10  
```





### Service与Ingress

问题：LoadBalancer的Service 需要给每一个Service都创建负载均衡服务

解决：内置一个全局的负载均衡服务 给不同的后端Service使用

Ingress就是Service的Service，是k8s对反向代理的抽象

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      # 每一个path都对应一个后端
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```





## 作业调度与资源管理



### 资源模型与资源管理

**资源模型**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

Pod：

- CPU：可压缩资源 资源不足时 pod会饥饿 但是不会退出
- 内存：不可压缩资源 资源不足时 pod会因为OOM被内核杀掉

- request：

  用于调度（动态资源边界）

- limits：

  用于cgroups限制



**QoS**：用于宿主机资源回收时 对Pod进行eviction时使用

- guranteed：requests == limits
- burstable：只要有一个container设置了requests级别
- bestEffort：requests和limits都没有



**cpuset设置**

将容器绑定到某个CPU的核上 减少CPU上下文切换的次数

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
    # cpu的requests和limits设置为同一个相等的整数值 该pod就会被绑定在两个独占的CPU核上
      limits:
        memory: "200Mi"
        cpu: "2"
      requests:
        memory: "200Mi"
        cpu: "2"
```





### 默认调度器

为新创建的出来的Pod寻找一个最合适的Node

挑选过程：

1. 调用predicate调度算法检查Node是否符合条件
2. 调用priority调度算法 给上一步的每个Node打分 选取分最高的Node
3. 将pod的spec.nodeName填上选中的Node名字



调度器核心：两个控制循环

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202303280733037.jpg)

- Informer Path：

  启动informer 来监听etcd中pod node service相关api对象的变化

- scheduling path：

  负责pod的具体调度





### 默认调度器调度策略解析

缓存化调度器中关于集群和Pod的信息

**Predicates**

实际计算时 会启动16个协程 来并发为集群中所有Node计算predicates

分类：

- generalPredicates：

  过滤CPU 内存 端口 是否为指定节点 是否满足pod的需求

- 与volume相关的

- 与宿主机相关的

  检查宿主机污点 是否内存不足

- 与Pod相关的

  检查待调度pod与Node上已有pod之间的关系



**Priorites**

给Node打分 0-10

- 选择空闲资源最多的宿主机
- 选择调度完成后 节点中各种资源分配最均衡的节点
- pod镜像已经存在Node上





### 默认调度的优先级与抢占机制

> 解决的问题：Pod调度失败时应该怎么办

正常情况：pod调度失败 会等待pod更新或者集群状态变化 进行再次调度

优先级高的pod：pod调度失败 会挤走Node上优先级低的pod



```yaml
# 定义PriorityClass
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
# 值越高代表优先级越大  
value: 1000000
globalDefault: false
description: "This priority class should be used for high priority service pods only."



apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```



**优先级**：优先级高的Pod会优先出队

**抢占**：

高优先级Pod调度失败后 会在集群中找一个Node 将低优先级Pod删除后 将Pod调度到这个节点上

抢占发生时 pod不会立即调度到被抢占的Node上 而是会在pod上设置spec.nominatedNodeName 在下一个调度周期决定调度（因为删除低优先级pod需要30s 这段时间内集群情况可能发生变化 更高优先级pod也可能出现）



机制设计：

1. 确认被抢占的Node和被删除的Pod列表

2. 清理被删除的pod的nominatedNodeName

3. 将抢占者的nominatedNodeName设置为选中的Node

   （这里会让pod从unschedulableQ队列进入activeQ队列）

   > activeQ：下一个调度周期中需要调度的Pod
   >
   > unschedulableQ：用来存放调度失败的Pod
   >
   > 关键：当unschedulableQ中的Pod更新后 会将这个pod移动到activeQ

4. 开启一个协程 同步地删除牺牲者

5. 在正常的调度流程中调度抢占者 对被指定抢占的Node做predicates时 需要做两遍 第一遍假设抢占成功 第二遍忽略 仅当两个都通过才调度





### k8s GPU管理与Device Plugin

GPU相关信息放在创建容器的CRI中



**Device Plugin机制**

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202304031934392.jpg)



1. 硬件设备对应的Device Plugin定期向kubelet汇报该Node上GPU的信息
2. kubelet拿到列表后 在其发往APIServer的心跳中 以Extended Resource的方式 加上GPU的数量
3. 被调度的pod 会被kubelet分配一个gpu 调用device plugin的接口进行分配 同时可以拿到GPU的信息 
4. 创建容器的时候 CRI中就可以带上gpu信息



## 容器运行时





### SIG-Node与CRI

**kubelet**：在pod被调度到Node后 负责在宿主机上将Pod和Pod内的各个容器启动起来

工作原理

事件驱动的一个控制循环

- Pod更新事件
- Pod生命周期变化
- kubelet本身设置的执行周期
- 定时的清理事件

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202304031958028.png)



kubelet调用下层容器运行时：通过Container Runtime Interface 用于解耦  调用gPRC接口

k8s架构：

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202304032008632.png)



### CRI与容器运行时

CRI：

- 容器相关：

  创建和启动容器 删除容器 执行exec命令 

- 镜像相关：

  拉取和删除镜像



**CRI shim实现exec logs接口的方式**

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202304041947120.png)

kubectl exec的执行路径

1. api server
2. kubelet
3. CRI shim
4. kubelet
5. api server
6. CRI stream server（websocket server就能实现）





### Kata Containers与gVisor

> 解决的问题：
>
> docker容器只是进程 直接接触还是宿主机的硬件资源 

安全容器：

给进程分配了一个独立的操作系统内核 避免容器共享宿主机内核 避免容器进程夺取宿主机控制权的问题

- Kata Container：

  传统的轻量化的虚拟机 虚拟机本身就是一个Pod 

  共享Network和其他Namespace

  容器只能看到Guest Kernel和Hypervisor虚拟出来的硬件

- gVisor：

  使用一个进程为容器提供内核的能力：运行用户程序 执行系统调用

  借助KVM对系统调用进行拦截 交给Sentry处理 Sentry执行具体的系统调用

  



## 容器监控与日志

### Prometheus、Metrics Server与k8s监控体系

![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202304042021693.png)

核心：

主动去搜集被监控对象的Metrics数据(pull)，将数据存储在时间序列数据库中 之后可以按时间进行检索（也可以主动push）



Metrics类型：

- 宿主机的监控数据

  由Node上的一个DaemonSet提供 

  CPU 内存 磁盘 网络

- k8s组件的监控数据 如api server kubelet

  组件内工作队列长度 请求QPS 延迟

- k8s相关的监控数据（使用的是Metrics Server 将获取监控数据的请求url暴露出来 Metrics Server也是到prometheus中查询数据）

  Pod Node Container Service



```shell
# 通过下面的url 就可以访问到metrics server
http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/namespaces/<namespace-name>/pods/<pod-name>
```



![img](https://daxiao-img.oss-cn-beijing.aliyuncs.com/img/202304042031553.png)





### 容器日志收集

> 需要解决的问题：
>
> 容器 Pod Node挂掉的时候 应用的日志都可以被正常获取到

日志方案：

- Node上部署logging agent（放在单独的Pod中） 以DaemonSet的方式运行 宿主机上容器日志目录挂载进来（让这个DaemonSet可以访问到日志） 然后转发到到远端存储

  缺点：要求应用输出的日志 都是输出到标准输出和标准错误

- 当业务容器中的日志只能输出到文件中时 可以在一个sidecar容器中把这些日志文件重新输出到sidercar的标准输出和标准错误中 兼容第一种方案

  缺点：日志文件会在宿主机上存两份 

- 通过一个sidecar容器（放在应用Pod中） 直接把应用的日志文件发送到远程存储里面

注意事项：及时将日志文件从宿主机上清理掉 避免磁盘分区被打满









































