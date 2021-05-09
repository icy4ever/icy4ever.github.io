## Kafka安装并创建单Broker集群

#### 第一步：下载并解压

```bash
wget http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.3.0/kafka_2.12-2.3.0.tgz
tar -C kafka_2.12-2.3.0.tgz -xvf /usr/local/
```

#### 第二步：安装Java

```bash
wget https://download.oracle.com/otn/java/jdk/8u231-b11/5b13a193868b4bf28bcb45c792fce896/jdk-8u231-linux-x64.tar.gz

tar -C /usr/local/ -xvf /jdk-8u231-linux-x64.tar.gz\?AuthParam\=1575360766_069f914000c983c58a99bdafda123b34
```

##### 设置环境变量：

```bash
export JAVA_HOME=/usr/local/jdk1.8.0_231
export PATH=$GOPATH/bin:$JAVA_HOME/bin:$GOROOT/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:
```

#### 第三步：创建单节点的zk实例并启动Kafka

```
cd /usr/local/kafka_2.12-2.3.0/
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &
nohup bin/kafka-server-start.sh config/server.properties &
```

#### 第四步：创建Topic：

```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

如果没有问题 上述命令将输出 test

 ![img](https://images2018.cnblogs.com/blog/1228818/201805/1228818-20180507190731172-1317551019.png) 

至此，我们已经创建了一个broker和一个Topic：test

#### 第五步：向Topic发送一些信息(模拟生产者)：

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
>Hello World!
>My name is supkul.com.
```

#### 第六步：启动一个消费者：

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

如果没有问题，将显示刚刚在生产者发布的讯息。

[下一篇_Kafka Connect导入和导出数据](http://114.55.146.117/post.html?id=14)

