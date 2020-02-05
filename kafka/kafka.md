### Kafka

​		[中文文档](http://kafka.apachecn.org/)

#### 消息队列

##### 使用优点

* 解耦

  允许独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束

* 可恢复性

  系统一部分组件失效时不会影响到整个系统。一个处理消息的进程挂掉，加入队列的消息仍然可以在系统恢复后被处理

* 缓冲

  有助于处理和优化数据流经过系统的数据，解决生产信息和消费信息的处理速度不一致问题

* 灵活性&峰值处理能力

  使用消息队列能够使关键组件顶住突发的访问压力，不会因为突发的超负荷的请求而完全崩溃

* 异步通信

##### 消息队列的两种模式

* 点对点模式：消费者主动拉取，消息收到后会清除消息

* 发布/订阅模式     

  kafka是主动拉取模式，消费者消费数据之后不会清除消息

#### Zookeeper在kafka中的作用

​		Kafka集群中有一个broker会被选举为Controller，负责管理集群broker的上下线，所有topic的分区副本分配和leader选举工作，Conrtoller的管理工作都是依赖于zookeeper的

##### leader选举流程

![image-20200205122344902](image/image-20200205122344902.png)


#### Kafka架构

![image-20200203121038266](image/image-20200203121038266.png)

* Broker：一台kafka机器就是一个Broker，一个集群由多个broker组成，一个broker可以容纳多个topic
* Partition：为了实现扩展性，一个非常大的topic可以分布到多个broker上去，一个topic可以分为多个partition，每个partition都是一个有序的队列

##### Kafka工作流程

![image-20200203215153293](image/image-20200203215153293.png)

##### Kafka文件存储

![image-20200203220656714](image/image-20200203220656714.png)

​		由于 生产者生产的消息会不断地追加到log文件末尾，为防止log文件过大导致数据定位效率低下，kafka采用**分片和索引**机制，将每个partition分为多个segment，每个segment对应两个文件，“.index” 和 “.log”文件，这些文件位于一个文件夹下，该文件夹的命名规则为：topic名+分区序号，eg：first这个topic有三个分区，则对应文件夹为：first-0，first-1，first-2

​		index和log文件以当前segment 第一条消息的offset命名

1. **index文件和log文件**

![image-20200203221821976](image/image-20200203221821976.png)

#### Kafka生产者

##### 分区策略

为了方便在集群中扩展和提高并发

* 分区原则：

  将producer发送的数据要封装成一个ProducerRecord对象

  ![image-20200203223452539](image/image-20200203223452539.png)

  ​		在选择k-v形式的接口，将key的hash值与topic的pertition数进行取余得到partition的值

  ​		参数只有value时，第一次调用随机生成一个整数(后面每次调用都在这个整数上自增)，将这个值与topic可用的partition总数取余得到partition的值
  
  ```java
  producer.send(new ProducerRecord<Integer, String>(topic, message));
  ```

##### 数据可靠性保证

​		topic的每个partition收到producer发送的数据后，都需要向producer发送ack（确认收到），如果producer收到ack，就会进行下一轮的发送，否则会重新发送数据

##### acks参数设置

​		对于某些不重要的数据，即可靠性要求不高，能够容忍数据的少量丢失，没必要等ISR中的follower全部接收成功，kafka提供三种可靠性级别

* acks=0

  ​		producer不等待broker的ack，当broker故障时可能有**数据丢失**

* acks=1

  ​		producer等待broker的ack，partition的leader落盘成功后返回ack，如果在follower同步成功之前leader故障，会有**数据丢失**

* acks=-1、all

  ​		producer等待broker的ack，partition的leader和follower（ISR中的）全部落盘成功后返回ack，但是在follower同步完成后leader故障，则会发生**数据重复**

##### Exactly Once

​		数据不丢失且去重

​		kafka0.11版本引入“幂等性”，即就是producer无论向server发送多少次重复数据，server端只会持久化一条，**At Least Once + 幂等性 = Exactly Once**

​		启用幂等性，将Producer中的参数**enable.idompotence**设置为true，其实质就是将原来下游需要做的去重放在了数据上游。开启幂等性的Producer爱初始化的时候会被分配一个PID，发往同一个Partition 的消息会附带上Sequence Number。Broker端会对<PID，Partition，SeqNumber>做缓存，当具有相同主键 的消息提交时，Broker只会持久化一条

​		PID重启会发生变化，幂等性无法保证跨分区跨会话的Exactly Once

##### 写入方式

​		producer 采用推（push）模式将消息发布到 broker，每条消息都被追加（append）到分区（patition）中，属于**顺序写磁盘**（顺序写磁盘效率比随机写内存要高，保障 kafka 吞吐率：顺序写磁盘速度快是因为省去了大量磁头寻址的时间）。

##### 零复制技术

​		要复制或者移动一个file，原本要经过kernel space和user space，现在的话直接交给系统去管理，只需要经过kernel space即可

##### 生产者ISR

​		何时producer会收到ack，有两种方案：”半数以上的Follower完成同步就发送ack“和“全部Follower完成同步就发送ack”，kafka采用后者，虽然网络延迟高，但是在选举新的leader时，需容忍n台节点的故障，只需要n+1个副本，且网络延迟对kafka影响较小  

​		同步策略存在问题：所有的Follower开始同步数据，要是有一个Follower因为某种故障，迟迟不能与leader进行同步，那leader就要一直等下去，直到完成同步，才能发送ack

​		对同步策略进行优化，采用ISR：leader维护了一个动态的in-sync replica set，指和leader保持同步的Follower集合。当ISR中的Follower完成数据的同步之后，leader就发送ack，如果Follower长时间未向leader同步数据，则它会被ISR踢出集合，时间阈值由**replica.lag.time.max.ms**参数设定。leader发生故障后，就会从ISR中选举新的leader

##### 数据一致性问题

​		涉及到的故障处理细节，解决故障之后，leader及多个Follower之间数据不一致问题，否则会造成消费数据混乱。即log文件中的HW和LEO

* HW（high Watermark）

  ​		消费者能见到的最大的offset，ISR队列中最小的LEO

* LEO

  ​		每个副本最大的offset

* 存储数据一致性问题，leader发生故障

  ​		从ISR中选出一个新的leader，为保证多个副本之间的数据一致性问题，其余的Follower会先将各自的log文件高于HW那部分截掉，然后从新的leader同步数据

#### Kafka消费者

##### 分区分配策略

​		即确定那个partition由那个consumer来消费，有两种策略:

* RoundRobin 轮询

  当前消费者组中的所有消费者订阅的主题是一样的（实际中一个消费者组代表一个业务需求）

* Range（默认）

  按照单个topic来分配

  当消费者组里面的消费者个数发生改变的时候，会触发重新分配

##### offset的维护

​		由消费者group+topic+partition唯一确定一个offset，0.9版本开始，consumer默认将offset保存在kafka一个内置的topic中，该topic为：_consumer_offsets

​		修改comsumer配置文件即consumer.properties

```shell
#本人电脑配置文件路径：/usr/local/etc/kafa/consumer.properties
增加：exclude.internal.topics = false 除了内部topic
```

读取/消费offset

```shell
#在kafka目录下
kafka-console-consumer.sh _consumer_offsets --zookeeper 群名:端口号 --formatter ”kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter“ --consumer.config config/consumer.properties --from-begining
```

#### Kafka事务

​		0.11版本开始支持，事务可以保证Kafka在Exactly Once语义的基础上，生产和消费可以跨分区和跨会话，要么全部成功，要么全部失败

##### Producer事务

​		为了实现跨跨分区和跨会话的事务，需要引入一个全局唯一的TransactionID，并将producer获得的PID和TransactionID绑定，这样Producer重启后就可以通过正在进行的TransactionID获取原来的PID

​		为了管理TransactionID，kafka引入了一个新的组件Transaction Coordinator。Producer就是通过和Transaction Coordinator交互获得TransactionID对应的任务状态。Transaction Coordinator还负责将事务所有写入kafka的一个内部topic，这样即使重启整个服务，由于事务状态得到保存，进行中的事务状态可以得到恢复，从而继续运行

​		解决精准一次性写入到kafka集群

##### Consumer事务

​		对于Consumer而言，事务的保证就会相对较弱，尤其无法保证commit的信息被精确消费，这是由于Consumer可以通过offset访问任何信息，而且不同的segment file生命周期不同，同一事务的消息可能会在重启后出现被删除的情况

#### KafkaAPI

##### ProducerAPI

​		发送消息采用是是**异步发送**，发送过程有两个线程：main线程和sender线程，以及一个线程共享变量-RecordAccumulator，sender线程不断从RecordAccumulator中拉取消息发送到kafka Borker

![image-20200205185454285](image/image-20200205185454285.png)

```java
package kafka;

import java.util.Properties;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
/**
 * @author jiabiying <jiabiying@kuaishou.com>
 * Created on 2020-01-16
 */
public class ProducerDemo extends Thread {
    private final KafkaProducer<Integer, String> producer;
    private String topic;

    public ProducerDemo(String topic) {
        //设置生产者的配置信息
        Properties properties = new Properties();
        //Kafka地址
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        //kafka客户端demo
        properties.put(ProducerConfig.CLIENT_ID_CONFIG, "ProducerDemo");
        //发送端消息确认模式
        properties.put(ProducerConfig.ACKS_CONFIG, "-1");
        //设置key的序列化
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.IntegerSerializer");
        //设置value的序列化
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
      //添加自定义分区器
        properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "kafka.parditioner.ParditionDemo");
        producer = new KafkaProducer<>(properties);
        this.topic = topic;
    }

    @Override
    public void run() {
            int num = 0;
            while (num < 100) {
                String message = "message--->" + num;
              	//有回调函数
                producer.send(new ProducerRecord<>(topic, message), (recordMetadata, e) -> {
                    if(e == null) {
                        System.out.println("分区：" + recordMetadata.partition() + "offset：" +recordMetadata.offset());
                    }
                });
                num++;
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
//            //关闭资源
//            producer.close();
    }

    public static void main(String[] args) {
        new ProducerDemo("test").start();
    }
}
```

自定义分区器

```java
import java.util.Map;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;

/**
 * @author jiabiying <jiabiying@kuaishou.com>
 * Created on 2020-02-05
 */
//自定义分区器
public class ParditionDemo implements Partitioner {
    //序列化之后
    @Override
    public int partition(String topic, Object key, byte[] bytes, Object value, byte[] bytes1, Cluster cluster) {
        //根据业务需求进行更改逻辑
        Integer partitionSize = cluster.partitionCountForTopic(topic);
        return key.toString().hashCode() % partitionSize;
    }

    @Override
    public void close() {
    }

    @Override
    public void configure(Map<String, ?> map) {
    }
}

```

##### ConsumerAPI

```java
import java.util.Collections;
import java.util.Properties;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.ProducerConfig;

/**
 * @author jiabiying <jiabiying@kuaishou.com>
 * Created on 2020-01-16
 */
public class ConsumerDemo extends Thread{

    public static void main(String[] args) {
        KafkaConsumer<Integer, String> kafkaConsumer;
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:2181");
        //消费者组
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "ConsumerDemo");
        //重置消费者的offset(生效满足条件：消费组发生变化，以前的数据过期被删)
        properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        //消费者消费消息以后自动提交offset，只有当消息提交以后，该消息才不会被再次接收到;也可以通过consumer.commitSync()的方式实现手动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        //自动提交的延时
        properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        //反序列化 key
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.IntegerDeserializer");
        //反序列化 value
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        //构建 KafkaConsumer
        kafkaConsumer = new KafkaConsumer<>(properties);
        //订阅 topic，一个或多个
        kafkaConsumer.subscribe(Collections.singletonList("test"));
        while (true) {
            //拉取消息
            ConsumerRecords<Integer, String> consumerRecord = kafkaConsumer.poll(100);
            //解析并打印ConsumerRecords
            for (ConsumerRecord<Integer, String> record : consumerRecord) {
                System.out.println("ConsumerDemo receive {" + record.value() + "}");
            }
        }
    }
}
```

**手动提交offset**

​		方法有两种：commitSync（同步提交）和 commitAsync（异步提交），两者都会将本次poll的一批数据最高的偏移量提交

​		不同的是：commitSync会阻塞当前线程，一直到提交成功，并且会自动失败重试；而commitAsync则没有失败重试机制，有可能提交失败

```java
/*
commitSync:它可靠，效率低
*/
//首先关掉自动提交
properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
//再消费完数据之后，调用commitSync
kafkaConsumer.commitSync();

/*
commitAsync:不可靠，容易丢失数据，效率高
*/
 kafkaConsumer.commitAsync((map, e) -> {
                if(e == null) {
                    System.out.println(map);
                }
 });
```

​		commitAsync为了尽量少丢数据，采用延时提交，但是又会出现数据重复；commitSync在某些特殊情况下也会出现数据重复，so出现了自定义存储offset，我们可以将offset存在mysql，将数据消费处理和offset入库放到一个事务里面，保证数据不会丢失也不会去出现数据重复的场景

​		但是offset维护还是相当麻烦，要考虑到消费者的Rebalance即重新分配，就是当有新的消费者加入、已有消费者推出或者所订阅的主题的分发发生变化，就会触发到分区的重新分配

​		消费者发生Rebalance之后，每个消费者消费的分区就会发生改变，so消费者首先要获取到自己被重新分配的分区，并且定位到每个分区最近提交的offset位置继续消费

```java
//订阅 topic，一个或多个
        kafkaConsumer.subscribe(Collections.singletonList("test"), new ConsumerRebalanceListener() {
            //该方法会在Rebalance之前调用
            @Override
            public void onPartitionsRevoked(Collection<TopicPartition> collection) {
                //提交当前的offset
                //commitOffset(currentOffset);
            }
            //该方法会在Rebalance之后调用
            @Override
            public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
                //currentOffset.clear();
                for(TopicPartition partition : partitions) {
                    kafkaConsumer.seek(partition, getOffset(partition));
                }
            }
        });

		//获取某分区的最新offset
    private static long getOffset(TopicPartition partition) {
        return 0;//自己实现
    }

    //提交该消费者所有分区的offset
    private static void commitOffset(Map<TopicPartition, Long> currentOffset) {
   			//自己实现
    }
```

##### 自定义拦截器

​		0.10版本引入，主要用于clients端的定制化控制逻辑

eg：在发送的消息value之前添加时间戳，发送消息成功之后更新成功消息数或失败消息数![image-20200206002400975](image/image-20200206002400975.png)

```java
  //添加拦截器
  ArrayList<String> interceptors = new ArrayList<>();
  interceptors.add("kafka.interceptor.ProducerTimeInterceptorDemo");
  interceptors.add("kafka.interceptor.ProducerCountInterceptorDemo");
  properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptors);
  
//自定义时间拦截器
public class ProducerTimeInterceptorDemo implements ProducerInterceptor<String, String> {
    @Override
    public ProducerRecord onSend(ProducerRecord<String, String> producerRecord) {
        //创建一个新的ProducerRecord对象,更改value值(添加时间戳)，返回
        return new ProducerRecord<>
                (producerRecord.topic(), producerRecord.partition(), producerRecord.key(), System.currentTimeMillis() + "," + producerRecord.value());
    }
    //其余方法省略
}

//自定义数量统计拦截器
public class ProducerCountInterceptorDemo implements ProducerInterceptor {
    private int success;
    private int error;

    @Override
    public ProducerRecord onSend(ProducerRecord producerRecord) {
        return producerRecord;
    }

    @Override
    public void onAcknowledgement(RecordMetadata recordMetadata, Exception e) {
        if(recordMetadata != null) {
            success++;
        } else {
            error++;
        }
    }

    @Override
    public void close() {
        System.out.println("success：" + success);
        System.out.println("error：" + error);
    }
  	//其余方法省略
}
```

#### Kafka监控 Eagle

​		开源的 Kafka 集群监控系统

##### 修改kafka启动命令

```shell
#将原来的代码 kafka堆的设置：
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi
#修改为
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
		export KAFKA_HEAP_OPTS=“-server -Xmx2G -Xms2G -XX:PermSize=128m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=5 -XX:InitiatingHeapOccupancyPercent=70”
    export JMX_PORT="9999"
    #export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi
```

##### 下载安装eagle

​		[安装步骤及使用](https://blog.csdn.net/qq_19524879/article/details/82848797?utm_source=blogxgwz9)

```shell
#配置其环境变量：
export KE_HOME=路径
export PATH=$PATH:$KE_HOME/bin

#使用，进入bin
chmod 777 ke.sh
#修改conf目录下的系统配置
```

####  安装&练习

1. 安装：brew install kafka

   ​           默认位置：/usr/local/Cellar/kafka       配置文件路径：/usr/local/etc/kafa/server.properties

2. 启动：

   ```java
   zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties
   kafka-server-start /usr/local/etc/kafka/server.properties
   //不会显示启动日志  
   kafka-server-start -daemon /usr/local/etc/kafka/server.properties  
   ```

3. 创建单区的topic即test

   ```java
   kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
   //查看topic
   kafka-topics --list --zookeeper localhost:2181
   ```

4. 创建生产者：

   ```java
   kafka-console-producer --broker-list localhost:9092 --topic test
   或者
   kafka-console-producer.sh --topic test --broker-list 集群名:9092     
   ```

5. 创建消费者

   ```java
   //从头开始消费
   kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
   或者
   //从连接时开始消费  
   kafka-console-consume.sh --topic test --bootstrap-server 集群名:9092    
   ```

6. 启动消费者（同一group下多个消费者）

```shell
#修改consumer.properties配置
group.id=组名

kafka-console-consume.sh --topic test --bootstrap-server 集群名:9092 --consumer.config
config/consumer.properties
或者
kafka-console-consume.sh --zookeeper 集群名:端口号 --topic test --consumer.config
config/consumer.properties
```

##### 修改配置文件

```shell
# broker的全局唯一编号，不能重复
broker.id=0
#处理网络请求的线程数量  
num.network.threads=3
#处理磁盘io的线程数量
num.io.threads=8
#发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
#请求套接字的缓冲区大小
socket.request.max.bytes=104857600
# kafka运行日志的存放路径（暂存数据）
log.dirs=/usr/local/var/lib/kafka-logs
# the brokers.topic在当前brokers上的分区数
num.partitions=1
#用来恢复和清理data下数据的线程数
num.recovery.threads.per.data.dir=1  
# segment文件保留的最长时间，超时将被删除
log.retention.hours=168
# 配置连接zookeeper集群地址
zookeeper.connect=localhost:2181  

#分发文件  
使用xsync filename就能将filename分发到集群中的各个节点中  
```



##### 启动脚本-群起群关

```shell
#!/bin/bash

case $1 in
"start"){
		for i in 群1 群2 群3...
		do
		  #全路径
			ssh $i "kafka-server-start /usr/local/etc/kafka/server.properties"
		done
};;

"stop"){
		for in 群1 群2 群3...
		do
			ssh $i "kafka-server-stop"
		done
};;
esac

保存文件，并给予权限chmod 777 文件名.sh
```

#### 相关命令

* 查看某个集群的topic

  kafka-topics.sh --list --zookeeper 集群名:端口号 

* 创建topic

  kafka-topics.sh --create --zookeeper 集群名:端口号  --topic 主题名 --partitions 2 --replication-factor 2

* 删除topic

  kafka-topics.sh --delete --zookeeper 集群名:端口号  --topic 主题名 

* 某个topic的详细信息

  kafka-topics.sh --describe --topic 主题名  --zookeeper 集群名:端口号  

### Kafka-streams

**WordCountApplication**

```java
package kafka;

import java.util.Arrays;
import java.util.Locale;
import java.util.Properties;
import java.util.regex.Pattern;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.common.utils.Bytes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.state.KeyValueStore;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Produced;
import org.apache.kafka.streams.StreamsBuilder;

/**
 * @author jiabiying <jiabiying@kuaishou.com>
 * Created on 2020-01-17
 */

public class WordCountApplication {

    public static void main(final String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "kafka-word-count");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        //使用默认的拓扑从word-count-input-topic中创建源
        KStream<String, String> textLines = builder.stream("word-count-input-topic");
        KTable<String, Long> wordCounts = textLines
                .flatMapValues(textLine -> Arrays.asList(textLine.toLowerCase().split("\\W+")))
                .groupBy((key, word) -> word)
                .count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>> as("counts-store"));
        wordCounts.toStream().to("word-count-output-topic", Produced.with(Serdes.String(), Serdes.Long()));

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```

* 解析

  ```java
  <KR, VR> KStream<KR, VR> flatMap(KeyValueMapper<? super K, ? super V, ? extends Iterable<? extends KeyValue<? extends KR, ? extends VR>>> var1);
  
   <VR> KStream<K, VR> flatMapValues(ValueMapper<? super V, ? extends Iterable<? extends VR>> var1);
  
  <VR> KStream<K, VR> flatMapValues(ValueMapperWithKey<? super K, ? super V, ? extends Iterable<? extends VR>> var1);
  ```

  K，V：这一对是指原始流中的键值对；
  KR，VR：这一对指的是结果流中的键值对。
          flatMap是将输入流中的每一条记录转成输出流中的0个或者多个记录。例如，输入流中的一条记录<K,V>经过flatMap后变成<K':V'>, <K'':V''>, ...成为了输出流中的多条记录。

  ​        而flatMapValues则不同，flatMapValues是将输入流中的一条记录转成输出流中有着相同Key值的0个或多个记录。例如，输入流中的一条记录<K,V>经过flatMapValues后变成了<K:V'>, <K:V''>, ...成为了输出流中相同K的多条记录

  ​		ValueMapper是将一个value映射成一个新的value；

  ​		KeyValueMapper是将一个key-value键值对映射成一个新的key-value键值对，其中的key和value都是可以修改的

  ​		ValueMapperWithKey是将一个value映射成一个新的value与valueMapper的区别是ValueMapperWithKey会在apply方法中带着只读的Key

  ​		groupBy和count，就类似SQL中的操作一样。groupBy操作会重新组织流中的数据，并且使用KeyValueMapper提供的值作为新Key，这个过程中Kafka Streams会在kafka cluster上生成一个叫做${applicationId}-XXX-repartition的新topic。最后count方法会根据组织好的key统计流中的出现次数

* 步骤

1. 将其打成jar包
2. 启动kafka
3. 创建input topic
   kafka-topics.sh --zookeeper localhost:2181 --create --topic word-count-input-topic --partitions 1 --replication-factor 1
4. 为该topic生成一些文本
   bin/kafka-console-producer.sh --broker-list localhost:9092 --topic word-count-input-topic
5. 运行jar
   java -cp ***.jar  全限类名
6. consumer查看结果
   bin/kafka-console-consumer.sh --topic word-count-output-topic --from-beginning --bootstrap-server localhost:9092 --property print.key=true

### Kafka源码部分

#### Partitoin分区策略

​		生产者，当未指定分区时：

```java

public class DefaultPartitioner implements Partitioner {
    private final ConcurrentMap<String, AtomicInteger> topicCounterMap = new ConcurrentHashMap();

    public DefaultPartitioner() {
    }

    public void configure(Map<String, ?> configs) {
    }
		//分区策略的实现
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
      	//拿到集群的所有分区数
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
      	//是否指定了key值
        if (keyBytes == null) {
            int nextValue = this.nextValue(topic);
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return ((PartitionInfo)availablePartitions.get(part)).partition();
            } else {
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
		//每次自增
    private int nextValue(String topic) {
        AtomicInteger counter = (AtomicInteger)this.topicCounterMap.get(topic);
        if (null == counter) {
            counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
            AtomicInteger currentCounter = (AtomicInteger)this.topicCounterMap.putIfAbsent(topic, counter);
            if (currentCounter != null) {
                counter = currentCounter;
            }
        }

        return counter.getAndIncrement();
    }

    public void close() {
    }
}
```

#### 拦截器原理

​		对于Producer而言，interceptor使得用户在发送消息之前及producer回调逻辑前有机会对消息做一些定制需求，也可以形成一个拦截器链

​		interceptor的实现接口是ProducerInterceptor

```java
public interface ProducerInterceptor<K, V> extends Configurable {
    //该方法封装进KafkaProducer.send方法中，即它运行在用户主线程中，Producer确保在消息被序列化之前以及计算分区前调用该方法
    ProducerRecord<K, V> onSend(ProducerRecord<K, V> var1);
		//该方法会在消息从RecordAccumulator成功发送到kafka broker之后，或者在发送过程中失败时调用。通常都是在producer回调逻辑触发之前。onAcknowledgement运行在producer的IO线程中，不建议在此方法中放入很重的逻辑，否则会导致producer发送消息的效率变慢
    void onAcknowledgement(RecordMetadata var1, Exception var2);
    //关闭拦截器，清理资源等
    void close();
}

//Configurable类的方法
public interface Configurable {
  	//获取配置信息和初始化数据时调用
    void configure(Map<String, ?> var1);
}
```