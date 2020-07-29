# Kafka配置

\[TOC\]

## 概念

### broker

Broker是kafka实例，每个服务器上有一个或多个kafka的实例，我们姑且认为每个broker对应一台服务器。每个kafka集群内的broker都有一个不重复的编号。

### topic

消息的主题，可以理解为消息的分类，kafka的数据就保存在topic。在每个broker上都可以创建多个topic。

### partition

Topic的分区，每个topic可以有多个分区，分区的作用是做负载，提高kafka的吞吐量。同一个topic在不同的分区的数据是不重复的，partition的表现形式就是一个一个的文件夹！

### Replication

每一个分区都有多个副本，副本的作用是做备胎。当主分区（Leader）故障的时候会选择一个备胎（Follower）上位，成为Leader。在kafka中默认副本的最大数量是10个，且副本的数量不能大于Broker的数量，follower和leader绝对是在不同的机器，同一机器对同一个分区也只可能存放一个副本。

### Consumer Group

我们可以将多个消费组组成一个消费者组，在kafka的设计中同一个分区的数据只能被消费者组中的某一个消费者消费。同一个消费者组的消费者可以消费同一个topic的不同分区的数据，这也是为了提高kafka的吞吐量，但是不会组内多个消费者消费同一分区的数据

### Zookeeper

kafka集群依赖zookeeper来保存集群的的元信息，来保证系统的可用性。

### offset

offset是一个占8byte的有序id号，它可以唯一确定每条消息在parition内的位置，在早期的版本中，消费者将消费到的offset维护zookeeper中，consumer每间隔一段时间上报一次，这里容易导致重复消费，且性能不好，在新的版本中消费者消费到的offset已经直接维护在kafk集群的\_\_consumer\_offsets这个topic中。 offset更新的方式，大致分为两类：

* 自动提交，设置enable.auto.commit=true，更新的频率根据参数【auto.commit.interval.ms】来定。这种方式也被称为【at most once】，fetch到消息后就可以更新offset，无论是否消费成功。
* 手动提交，设置enable.auto.commit=false，这种方式称为【at least once】。fetch到消息后，等消费完成再调用方法【consumer.commitSync\(\)】，手动更新offset；如果消费失败，则offset也不会更新，此条消息会被重复消费一次。

  **存储策略**

  无论消息是否被消费，kafka都会保存所有的消息。那对于旧数据有什么删除策略呢？

  1、 基于时间，默认配置是168小时（7天）。

  2、 基于大小，默认配置是1073741824。

  需要注意的是，kafka读取特定消息的时间复杂度是O\(1\)，所以这里删除过期的文件并不会提高kafka的性能。

## 安装kafka

### 下载配置

kafka是基于java的分布式程序，因此我们需要在系统环境中配置jdk，通过命令`java --version`可以查看是否安装了Java。 下载地址： [https://www.apache.org/dyn/closer.cgi?path=/kafka/2.3.0/kafka\_2.11-2.3.0.tgz](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.3.0/kafka_2.11-2.3.0.tgz) 下载后解压，服务器的配置文件路径为：kafka-2.3.0-src/config/server.properties 在此配置文件中可以配置broker id以及log路径，默认保存策略\(log.retention.hours=168\)等。 MAC下可以直接通过brew安装：`brew install kafka` 配置文件位置：

```text
/usr/local/etc/kafka/server.properties
/usr/local/etc/kafka/zookeeper.properties
```

### 启动

先启动zookeeper的发现协议：

```text
➜ kafka_2.11-2.3.0 ./bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
```

然后启动kafka：

```text
➜ kafka_2.11-2.3.0 ./bin/kafka-server-start.sh -daemon config/server.properties
```

### 创建主题

如果往不存在的topic写数据，kafka会自动创建topic，分区和副本的数量根据默认配置都是1。 也可以通过命令行创建：

```text
➜ kafka_2.11-2.3.0 bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
Created topic test.
```

此时在log下面就可以看到相关topic了：

```text
➜ kafka_2.11-2.3.0 ls /Volumes/DATA/kafka_log/test-0
00000000000000000000.index 00000000000000000000.log 00000000000000000000.timeindex leader-epoch-checkpoint
```

也可以用命令直接查看到：

```text
➜ kafka_2.11-2.3.0 bin/kafka-topics.sh --list --zookeeper localhost:2181
test
```

还可以查看topic的整体信息：

```text
➜ kafka_2.11-2.3.0 bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
Topic:test PartitionCount:1 ReplicationFactor:1 Configs:
 Topic: test Partition: 0 Leader: 1 Replicas: 1 Isr: 1
```

### 分组信息

当使用kafka消费时，如果没有对应分组会自动创建分组：

```text
➜ kafka_2.11-2.3.0 bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group test_group


GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID
test_group test 0 204 204 0 ___TestGroup_in_github_com_eager7_kafka@plainchant-desktop.lan (github.com/segmentio/kafka-go)-4b3d827c-2d5e-4ae6-a5cd-9ba31e233a10 /192.168.199.216 ___TestGroup_in_github_com_eager7_kafka@plainchant-desktop.lan (github.com/segmentio/kafka-go)
```

分组游标是kafka维护的，不能手动设置。

