## ssh保持登录

在服务端执行以下命令：

```bash
vim /etc/ssh/ssh_config
```

在底部增加两行

```bash
ClientAliveInterval 60
ClientAliveCountMax 10
```

在客户端执行（mac OS 为例）:

```bash
vim /etc/ssh/ssh_config
```

在底部增加两行

```bash
TCPKeepAlive=yes
ServerAliveInterval 60
ServerAliveCountMax 3
```

