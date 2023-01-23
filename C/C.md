# C

[TOC]

**gdb**

```shell 
# 下载与安装
sudo yum -y install centos-release-scl
sudo yum-config-manager --enable rhel-server-rhscl-7-rpms
sudo yum -y install devtoolset-7
scl enable devtoolset-7 bash

# 编译
gcc -g hello.c

gdb a.out

set args 指定的命令行运行参数

b 指定行数打断点

r 运行程序

l 显示下面10行

n step over 不进入函数
 
s step into 进入函数

p 打印内容

q 退出

c 执行直到断点或者结束

info b 查看所有断点信息

disable b 取消断点

enable b 重新启用断点

set variables i = 100 # 设置变量参数
```



[代码规范](https://users.ece.cmu.edu/~eno/coding/CCodingStandard.html)

