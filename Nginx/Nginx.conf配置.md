## Nginx.conf 配置

```bash
user nginx;#表明用户,配置错误可能产生 403 forbidden
events {
        worker_connections  1024;
}
http{
        upstream backend {#负载均衡
                least_conn;#这行代表使用的负载均衡方法，没有配置就是Round robin,
#least_conn 根据权重和最少连接数来均衡连接
#least_time 根据响应时间o分配连接
#hash
#random     随机分配
#ip_hash    根据ip来hash分配
                        server 114.55.146.117  weight=1;#根据权重请求将被分发到配置的服务器上
                                server 114.55.146.117  weight=2;
        }
        server{#服务器配置块
                listen 80;#监听80端口
                        location / {
                                proxy_pass http://114.55.146.117:8080; #反向代理 请求80端口的时候会代理转发到8080端口 外部感受不到差异
                        }
        }
        server{
                listen 8080;
                location / {
                        root ./blog;
                        index index.html;
                }
        }
}
```

