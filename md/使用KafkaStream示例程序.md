## 使用Kafka Stream示例程序

介绍：这个程序是用来流式计算输入的语句的各个单词出现的次数。

和[Kafka安装并创建单个集群](http://114.55.146.117/post.html?id=13)一样先创建 zk并启动Kafka Server

```
cd /usr/local/kafka
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &
nohup bin/kafka-server-start.sh config/server.properties &
```

创建两个topic 

```
bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-plaintext-input
bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-wordcount-output \
    --config cleanup.policy=compact
```

查看topic是否创建成功

```
bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe
 //将输出以下
Topic:streams-plaintext-input   PartitionCount:1    ReplicationFactor:1 Configs:
    Topic: streams-plaintext-input  Partition: 0    Leader: 0   Replicas: 0 Isr: 0
Topic:streams-wordcount-output  PartitionCount:1    ReplicationFactor:1 Configs:cleanup.policy=compact
    Topic: streams-wordcount-output Partition: 0    Leader: 0   Replicas: 0 Isr: 0
```

启动计算单词个数的demo程序

```
nohup bin/kafka-run-class.sh org.apache.kafka.streams.examples.wordcount.WordCountDemo &
```

另起一个命令行终端 在第一个输入以下：

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic streams-plaintext-input
all streams lead to kafka
hello kafka streams
```

另一个终端将显示

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic streams-wordcount-output \
    --from-beginning \
    --formatter kafka.tools.DefaultMessageFormatter \
    --property print.key=true \
    --property print.value=true \
    --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
    --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
 
all     1
streams 1
lead    1
to      1
kafka   1
hello   1
kafka   2
streams 2
```

具体增加过程如下：

 <img src="http://kafka.apache.org/23/images/streams-table-updates-01.png" alt="img" style="zoom: 50%;" />  <img src="http://kafka.apache.org/23/images/streams-table-updates-02.png" alt="img" style="zoom:50%;" /> 