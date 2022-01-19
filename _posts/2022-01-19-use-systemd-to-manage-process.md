# 如何使用systemd托管应用程序
## systemd介绍
systemd是一个Linux系统初始化和管理的软件，现在已经成为了Linux发行版本的新标准。
## systemctl
systemctl类似于k8s的ctl，都是用来提供一组命令来管理软件的。
## 如何托管程序
一般来说，假设我们有一个可执行程序main，他位于/root目录下。
我们需要运行该程序时，需要执行如下命令
```bash
$ nohup /root/main > output &
```
当我们需要重启程序时，往往需要lsof通过端口查询对应进程，然后kill掉。这种操作太过原始，也太过麻烦。
使用systemd后就可以通过一些简单的命令来管理程序。
### 定义.service文件
```
[Unit]
Description=xxx daemon

[Service]
ExecStart=/root/main #这里定义你的可执行文件的命令
IgnoreSIGPIPE=false
KillMode=process
Restart=on-failure
```
### 启动
```bash
systemctl start xxx.service
```
### 查看状态
```bash
systemctl status xxx.service
```
输出:

![企业微信截图_715211aa-2eba-40fa-8c50-cfd9cf64d1bf](https://user-images.githubusercontent.com/38686456/150084336-3c0fdb0b-3ddf-425e-802e-cfd5a7beb239.png)
### 停止
```bash
systemctl stop xxx.service
```
### 重启
``` bash
systemctl restart xxx.service
```
