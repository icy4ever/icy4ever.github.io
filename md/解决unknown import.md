## 解决unknown import path "golang.org/x/sys/unix": unrecognized import path "golang.org/x/sys"

实际上这种情况不要盲目导入包或者go get ，重新安装最新的go就可以解决：

```bash
wget https://dl.google.com/go/go1.13.4.linux-amd64.tar.gz
tar -C /usr/local -xvf go1.13.4.linux-amd64.tar.gz
```

顺带贴一下/etc/profile配置文件：

```
export GOPATH=/root/go
export GIT_SSL_NO_VERIFY=1
export PATH=$GOPATH/bin:$GOROOT/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:
export GOROOT=/usr/local/go
```

部署完项目后 需要更新一下依赖：

```bash
go mod vendor
```
