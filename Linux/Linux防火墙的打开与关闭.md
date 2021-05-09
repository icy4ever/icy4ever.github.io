## Linux 防火墙的打开关闭

### Centos防火墙相关

访问服务器端口的时候最好要提前检查一下是否开启对应端口，否则会导致5xx错误。

```bash
firewall-cmd --zone=public --add-port=9000/tcp --permanent	//打开9000端口
firewall-cmd --reload								//重启防火墙
firewall-cmd --list-all								//查看打开端口
```

### Ubuntu防火墙相关

```bash
ufw status											//查看防火墙状态
ufw reload											//防火墙重启
ufw allow 9000										//打开端口
```

防火墙打开之后若还是ping不通，那么可以查看下安全组相关端口是否开启。
