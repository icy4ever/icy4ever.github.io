## Kafka Connect导入和导出数据

首先启动两个独立模式运行的链接器 并且在后台运行（kafka根目录）

```
nohup bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties config/connect-file-sink.properties &
```

其次使用控制台后台监控查看该主题数据 并将其输出到stdout.txt文件中

```
nohup bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-test --from-beginning > stdout.txt &
```

写入数据到文件

```
echo Another line >> test.txt
```

查看结果：

```
vim stdout.txt
```

