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

​		**事务能够保证Kafka topic下每个分区的原子写入，实现原子的“读取-处理-写入”过程**（即如果某个应用程序在某个topic tp0的偏移量X处读取到了消息A，并且在对消息A进行了一些处理（如B = F（A））之后将消息B写入topic tp1，则只有当消息A和B被认为被成功地消费并一起发布，或者完全不发布时，整个读取过程写入操作是原子的。）

​		API要求事务性Producer的第一个操作应该是在Kafka集群中显示注册transactional.id。 当注册的时候，Kafka broker用给定的transactional.id检查打开的事务并且完成处理。 Kafka也增加了一个与transactional.id相关的epoch。Epoch存储每个transactional.id内部元数据。一旦这个epoch被触发，任何具有相同的transactional.id和更旧的epoch的Producer被视为僵尸，并被围起来, Kafka会拒绝来自这些Procedure的后续事务性写入。

##### 事务场景

1. 最简单的需求是producer发的多条消息组成一个事务这些消息需要对consumer同时可见或者同时不可见 
2. producer可能会给多个topic，多个partition发消息，这些消息也需要能放在一个事务里面，这就形成了一个典型的分布式事务
3. kafka的应用场景经常是应用先消费一个topic，然后做处理再发到另一个topic，这个consume-transform-produce过程需要放到一个事务里面，比如在消息处理或者发送的过程中如果失败了，消费位点也不能提交
4. producer或者producer所在的应用可能会挂掉，新的producer启动以后需要知道怎么处理之前未完成的事务 
5. 流式处理的拓扑可能会比较深，如果下游只有等上游消息事务提交以后才能读到，可能会导致rt非常长吞吐量也随之下降很多，所以需要实现read committed和read uncommitted两种事务隔离级别

##### 关键概念

1. 事务管理中事务日志是必不可少的，kafka使用一个内部topic来保存事务日志，这个设计和之前使用内部topic保存位点的设计保持一致。事务日志是Transaction Coordinator管理的状态的持久化，因为不需要回溯事务的历史状态，所以事务日志只用保存最近的事务状态
2. 因为事务存在commit和abort两种操作，而客户端又有read committed和read uncommitted两种隔离级别，所以消息队列必须能标识事务状态，这个被称作Control Message
3. producer挂掉重启或者漂移到其它机器需要能关联的之前的未完成事务所以需要有一个唯一标识符来进行关联，这个就是TransactionalId，一个producer挂了，另一个有相同TransactionalId的producer能够接着处理这个事务未完成的状态。注意不要把TransactionalId和数据库事务中常见的transaction id搞混了，kafka目前没有引入全局序，所以也没有transaction id，这个TransactionalId是用户提前配置的
4. TransactionalId能关联producer，也需要避免两个使用相同TransactionalId的producer同时存在，所以引入了producer epoch来保证对应一个TransactionalId只有一个活跃的producer epoch

##### Producer事务

​		为了实现跨跨分区和跨会话的事务，需要引入一个全局唯一的TransactionID，并将producer获得的PID和TransactionID绑定，这样Producer重启后就可以通过正在进行的TransactionID获取原来的PID

​		为了管理TransactionID，kafka引入了一个新的组件Transaction Coordinator。Producer就是通过和Transaction Coordinator交互获得TransactionID对应的任务状态。Transaction Coordinator还负责将事务所有写入kafka的一个内部topic，这样即使重启整个服务，由于事务状态得到保存，进行中的事务状态可以得到恢复，从而继续运行

​		解决精准一次性写入到kafka集群问题

```java
/*producer提供了五个事务方法：
initTransactions beginTransaction sendOffsets commitTransaction abortTransaction
*/

//事务id
properties.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "producer-1");// 每台机器唯一
properties.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);// 设置幂等性

producer = new KafkaProducer<>(properties);
        //初始化事务
        producer.initTransactions();
        try {
            Thread.sleep(1000);
            //开启事务
            producer.beginTransaction();
            int num = 0;
            while (num < 10) {
                String message = "message--->" + num;
                producer.send(new ProducerRecord<>("test", message), (recordMetadata, e) -> {
                    if(e == null) {
                        System.out.println("分区：" + recordMetadata.partition() + "offset：" +recordMetadata.offset());
                    }
                });
                num++;
            }
            //提交事务
            producer.commitTransaction();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            //终止事务
            producer.abortTransaction();
        }
        producer.close();

```

##### consume-Transform-Produce模式

```java
//Produce的properties配置如上例代码
//consume的properties的配置需增加：
properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
properties.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");//事务隔离级别

public static void main(String[] args) {
        // 创建生产者
        KafkaProducer<String, String> producer = new KafkaProducer<>(getProducerProperties());
        // 创建消费者
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(getConsumerProperties());
        // 初始化事务
        producer.initTransactions();
        // 订阅主题
        consumer.subscribe(Arrays.asList("cons-tran-prod"));
        for(;;){
            // 开启事务
            producer.beginTransaction();
            // 接受消息
            ConsumerRecords<String, String> records = consumer.poll(500);
            // 处理逻辑
            try {
                Map<TopicPartition, OffsetAndMetadata> commits = new HashMap<>();
                for(ConsumerRecord record : records){
                    // 处理消息
                    out.println("offset = " + record.offset()+ "，key =" + record.key() + "，value =" + record.value());
                    // 记录提交的偏移量
                    commits.put(new TopicPartition(record.topic(), record.partition()),new OffsetAndMetadata(record.offset()));
                    // 产生新消息
                    producer.send(new ProducerRecord<>("consumer-send", record.value() + "send"));
                }
                // 提交偏移量
                producer.sendOffsetsToTransaction(commits,"groupTest");
                // 事务提交
                producer.commitTransaction();

            }catch (Exception e){
                e.printStackTrace();
                //终止事务
                producer.abortTransaction();
            }
        }
```

##### Consumer事务

​		对于Consumer而言，事务的保证就会相对较弱，尤其无法保证commit的信息被精确消费，这是由于Consumer可以通过offset访问任何信息，而且不同的segment file生命周期不同，同一事务的消息可能会在重启后出现被删除的情况

​		Kafka保证Consumer最终只能提供非事务性消息或提交事务性消息。它将保留来自未完成事务的消息，并过滤掉已中止事务的消息。

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

##### 自定义分区器

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

#### Kafka事务原理

##### 事务协调器和事务日志

​		在Kafka 0.11.0引入事务Coordinator和事务日志

​		事务Coordinator是每个KafkaBroker内部运行的一个模块。事务日志是一个内部的Kafka Topic。每个Coordinator拥有事务日志所在分区的子集，即, 这些borker中的分区都是Leader。每个transactional.id都通过一个简单的哈希函数映射到事务日志的特定分区，事务日志文件__transaction_state-0。这意味着只有一个Broker拥有给定的transactional.id。通过这种方式，我们利用Kafka可靠的复制协议和Leader选举流程来确保事务协调器始终可用，并且所有事务状态都能够持久存储

​		事务日志只保存事务的最新状态而不是事务中的实际消息。消息只存储在实际的Topic的分区中。事务可以处于诸如“Ongoing”，“prepare commit”和“Completed”之类的各种状态中。正是这种状态和关联的元数据存储在事务日志中

##### 事务数据流

​		数据流在抽象层面上有四种不同的类型。

* producer和事务coordinator的交互

　　执行事务时，Producer向事务协调员发出如下请求：

1. initTransactions API向coordinator注册一个transactional.id。 此时，coordinator使用该transactional.id关闭所有待处理的事务，并且会避免遇到僵尸实例，由具有相同的transactional.id的Producer的另一个实例启动的任何事务将被关闭和隔离。每个Producer会话只发生一次
2. 当Producer在事务中第一次将数据发送到分区时，首先向coordinator注册分区
3. 当应用程序调用commitTransaction或abortTransaction时，会向coordinator发送一个请求以开始两阶段提交协议

* Coordinator和事务日志交互

　　随着事务的进行，Producer发送上面的请求来更新Coordinator上事务的状态。事务Coordinator会在内存中保存每个事务的状态，并且把这个状态写到事务日志中（这是以三种方式复制的，因此是持久保存的）

　　事务Coordinator是读写事务日志的唯一组件。如果一个给定的Borker故障了，一个新的Coordinator会被选为新的事务日志的Leader，这个事务日志分割了这个失效的代理，它从传入的分区中读取消息并在内存中重建状态

* Producer将数据写入目标Topic所在分区

　　在Coordinator的事务中注册新的分区后，Producer将数据正常地发送到真实数据所在分区。这与producer.send流程完全相同，但有一些额外的验证，以确保Producer不被隔离

* Topic分区和Coordinator的交互

1. 在Producer发起提交（或中止）之后，协调器开始两阶段提交协议。
2. 在第一阶段，Coordinator将其内部状态更新为“prepare_commit”并在事务日志中更新此状态。一旦完成了这个事务，无论发生什么事，都能保证事务完成
3. Coordinator然后开始阶段2，在那里它将事务提交标记写入作为事务一部分的Topic分区
4. 这些事务标记不会暴露给应用程序，但是在read_committed模式下被Consumer使用来过滤掉被中止事务的消息，并且不返回属于开放事务的消息（即那些在日志中但没有事务标记与他们相关联）
5. 一旦标记被写入，事务协调器将事务标记为“完成”，并且Producer可以开始下一个事务

##### 事务的相关配置

```shell
#Broker configs
#在ms中，事务协调器在生产者TransactionalId提前过期之前等待的最长时间，并且没有从该生产者TransactionalId接收到任何事务状态更新。默认是604800000(7天)
ransactional.id.timeout.ms：
#事务允许的最大超时：如果客户端请求的事务时间超过此时间，broke将在InitPidRequest中返回InvalidTransactionTimeout错误。这可以防止客户机超时过大，从而导致用户无法从事务中包含的主题读取内容
默认值为900000(15分钟)。这是消息事务需要发送的时间的保守上限。
max.transaction.timeout.ms
#事务状态topic的副本数量。默认值:3
transaction.state.log.replication.factor
#事务状态主题的分区数。默认值:50
transaction.state.log.num.partitions
#事务状态主题的每个分区ISR最小数量。默认值:2
transaction.state.log.min.isr
#事务状态主题的segment大小。默认值:104857600字节
transaction.state.log.segment.bytes

#Producer configs
enable.idempotence：开启幂等
#事务超时时间:事务协调器在主动中止正在进行的事务之前等待生产者更新事务状态的最长时间。
#默认是60000
transaction.timeout.ms
#用于事务性交付的TransactionalId。这支持跨多个生产者会话的可靠性语义，因为它允许客户端确保使用相同TransactionalId的事务在启动任何新事务之前已经完成。如果没有提供TransactionalId，则生产者仅限于幂等交付
transactional.id

#Consumer configs
isolation.level
read_uncommitted：以偏移顺序使用已提交和未提交的消息。
read_committed：仅以偏移量顺序使用非事务性消息或已提交事务性消息。为了维护偏移排序，这个设置意味着我们必须在使用者中缓冲消息，直到看到给定事务中的所有消息。
```

##### 事务性能以及如何优化

* Producer打开事务之后的性能

  事务如何执行：首先，事务只造成中等的写入放大，额外的写入在于：

1. 对于每个事务，我们都有额外的RPC向Coordinator注册分区

2. 在完成事务时，必须将一个事务标记写入参与事务的每个分区。同样，事务Coordinator在单个RPC中批量绑定到同一个Borker的所有标记，所以我们在那里保存RPC开销。但是在事务中对每个分区进行额外的写操作是无法避免的

3. 最后，我们将状态更改写入事务日志。这包括写入添加到事务的每批分区，“prepare_commit”状态和“complete_commit”状态

   ​        开销与作为事务一部分写入的消息数量无关，拥有更高吞吐量的关键是**每个事务包含更多的消息**

   ​        实际上，对于Producer以最大吞吐量生产1KB记录，每100ms提交消息导致吞吐量仅降低3％。较小的消息或较短的事务提交间隔会导致更严重的降级

   ​        增加事务时间的主要折衷是增加了端到端延迟。回想一下，Consume阅读事务消息不会传递属于公开传输的消息。因此，提交之间的时间间隔越长，消耗的应用程序就越需要等待，从而增加了端到端的延迟

* Consumer打开之后的性能

  Consumer在开启事务的场景比Producer简单得多，它需要做的是：

  1. 过滤掉属于中止事务的消息

  2. 不返回属于公开事务一部分的事务消息

  ​        因此，当以read_committed模式读取事务消息时，事务Consumer的吞吐量没有降低。这样做的主要原因是我们在读取事务消息时保持零拷贝读取

  ​       此外，Consumer不需要任何缓冲等待事务完成。相反，Broker不允许提前抵消包括公开事，so，Consumer是非常轻巧和高效的