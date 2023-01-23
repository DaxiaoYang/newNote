# Linux

查询所有自启动的服务

`systemctl list-unit-files | grep enable`



vi

```shell
# vi
# 命令模式下
ctrl + f 向下移动一页 
ctrl + b 向上移动一页
G 移动到文件最后一个行
gg 移动到文件第一行

x delete
dd 删除当前行
yy 复制当前行
p 粘贴
u 撤销

```



`git init`

```shell
echo "# learn" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/DaxiaoYang/designpattern.git
git push -u origin main
```



```shell
sed -i 's/old/new' textpath  #将textpath中old的内容替换为new 就是一个replace操作

# 计算cpu核数
cat /proc/cpuinfo | grep 'physical id' | sort | uniq | wc -l
```





[xshell快捷键](https://www.vckai.com/xshell-kuai-jie-jian-fei-chang-shi-yong)



[top命令](https://www.cnblogs.com/peida/archive/2012/12/24/2831353.html)

