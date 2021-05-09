## ssh无密码登录云服务器

在客户端输入以下命令（mac os 为例）：

```bash
ssh-keygen		//之后一直回车就可以了,生成相关公私钥
scp ../.ssh/id_rsa.pub root@123.4.5.6:/  //传输到远端根目录这里 ip和目录根据自己的情况来
```

在远端输入以下命令(ubuntu)：

```
cat /id_rsa.pub >> /root/.ssh/authorized_keys
```

将公钥内容放入ssh认证key中就可以了。