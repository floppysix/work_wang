#
bin/kafka-server-start.sh -daemon config/server.properties
# 0 虚拟机
1. 修改虚拟机的静态ip
```
# vim /etc/sysconfig/network-scripts/ifcfg-ens33

-----修改内容------
DEVICE=ens33
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=static 
NAME="ens33"
IPADDR=192.168.10.102
PREFIX=24
GATEWAY=192.168.10.2
DNS1=192.168.10.2
```
2. 查看Linux虚拟机的虚拟网络编辑器，编辑->虚拟网络编辑器->VMnet8![[Pasted image 20220703201450.png]]
![[Pasted image 20220703201526.png]]
3. 查看Windows系统适配器VMware Network Adapter VMnet8的IP地址![[Pasted image 20220703201616.png]]
4. 保证Linux系统ifcfg-ens33文件中IP地址、虚拟网络编辑器地址和Windows系统VM8网络IP地址相同
5. 修改主机名称
```
[root@hadoop100 ~]# vim /etc/hostname

hadoop102
```
7. 配置Linux克隆机主机名称映射hosts文件
```
[root@hadoop100 ~]# vim /etc/hosts

-------------添加如下内容-----------

192.168.10.100 hadoop100

192.168.10.101 hadoop101

192.168.10.102 hadoop102

192.168.10.103 hadoop103

192.168.10.104 hadoop104

```
9. 重启克隆机
```
[root@hadoop100 ~]# reboot
```
11. 修改windows的主机映射文件（hosts文件）
```
进入C:\Windows\System32\drivers\etc路径
打开hosts文件并添加如下内容，然后保存
192.168.10.100 hadoop100

192.168.10.101 hadoop101

192.168.10.102 hadoop102

192.168.10.103 hadoop103

192.168.10.104 hadoop104

```
12. 编写集群分发脚本xsync
	1. 在/home/atguigu/bin目录下创建xsync文件
		>[atguigu@hadoop102 opt]$ cd /home/atguigu
		>[atguigu@hadoop102 ~]$ mkdir bin
		>[atguigu@hadoop102 ~]$ cd bin
		>[atguigu@hadoop102 bin]$ vim xsync
```bash
#!/bin/bash

#1. 判断参数个数

if [ $# -lt 1 ]

then

    echo Not Enough Arguement!

    exit;

fi

#2. 遍历集群所有机器

for host in hadoop102 hadoop103 hadoop104

do

    echo ====================  $host  ====================

    #3. 遍历所有目录，挨个发送

    for file in $@

    do

        #4. 判断文件是否存在

        if [ -e $file ]

            then

                #5. 获取父目录

                pdir=$(cd -P $(dirname $file); pwd)

                #6. 获取当前文件的名称

                fname=$(basename $file)

                ssh $host "mkdir -p $pdir"

                rsync -av $pdir/$fname $host:$pdir

            else

                echo $file does not exists!

        fi

    done

done
```
	2. 修改脚本 xsync 具有执行权限
		[atguigu@hadoop102 bin]$ chmod +x xsync
	3. 将脚本复制到/bin中，以便全局调用
		[atguigu@hadoop102 bin]$ sudo cp xsync /bin/
	4. 同步环境变量配置（root所有者）
		[atguigu@hadoop102 ~]$ sudo ./bin/xsync /etc/profile.d/my_env.sh
	5. 让环境变量生效
		>[atguigu@hadoop103 bin]$ source /etc/profile
		>[atguigu@hadoop104 opt]$ source /etc/profile
		

13. SSH无密登录配置
```
		[atguigu@hadoop102 ~]$ ssh hadoop103

		[atguigu@hadoop102 .ssh]$ [atguigu@hadoop102 .ssh]$ ssh-keygen -t rsa

		[atguigu@hadoop102 .ssh]$ ssh-copy-id hadoop102
		[atguigu@hadoop102 .ssh]$ ssh-copy-id hadoop103
		[atguigu@hadoop102 .ssh]$ ssh-copy-id hadoop104
```

# 1 安装[[kafka]]

# 2 生产者
![[Pasted image 20220703205805.png]]

![[Pasted image 20220703210403.png]]

可以自定义分区器实现

**提高生产者吞吐量**
- 批次大小（默认16k）：batch.size
- 等待时间，修改为5-100ms：linger.ms
- 压缩类型（默认为none，修改为snappy）：compression.type
- 缓冲区大小（默认32m，修改为64m）：RecordAccumulator

**数据可靠性**——ack设置为-1（all）

**数据去重**
>幂等性：开启参数 enable.idempotence 默认为 true，false 关闭
>精确一次（Exactly Once） = 幂等性 + 至少一次（ ack=-1 + 分区副本数>=2 + ISR最小副本数量>=2）

**事务管理**
生产者自定义唯一的transactional.id

**数据有序**
开启幂等性后设置max.in.flight.requests.per.connection需要设置小于等于5


# 3 broker
![[Pasted image 20220703211948.png]]


# 4 消费者
![[Pasted image 20220703212820.png]]

![[Pasted image 20220703213051.png]]

![[Pasted image 20220703213148.png]]

![[Pasted image 20220703213438.png]]

![[Pasted image 20220703213453.png]]

![[Pasted image 20220703213517.png]]




# 5 与sringboot整合

## application.propertise配置
```
###########【Kafka集群】###########
# 指定 kafka 的地址
spring.kafka.bootstrap-servers=hadoop102:9092,hadoop103:9092,hadoop104:9092


###########【初始化生产者配置】###########
# 重试次数
spring.kafka.producer.retries=0
# 应答级别:多少个分区副本备份完成时向生产者发送ack确认(可选0、1、all/-1)
spring.kafka.producer.acks=1
# 批量大小
spring.kafka.producer.batch-size=16384
# 提交延时
spring.kafka.producer.properties.linger.ms=0
# 当生产端积累的消息达到batch-size或接收到消息linger.ms后,生产者就会将消息提交给kafka
# linger.ms为0表示每接收到一条消息就提交给kafka,这时候batch-size其实就没用了

# 生产端缓冲区大小(32m)
spring.kafka.producer.buffer-memory = 33554432
# Kafka提供的序列化和反序列化类
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
# 自定义分区器
#spring.kafka.producer.properties.partitioner.class=com.felix.kafka.producer.CustomizePartitioner

###########【初始化消费者配置】###########
# 默认的消费组ID
spring.kafka.consumer.group.id=defaultConsumerGroup
# 是否自动提交offset
spring.kafka.consumer.enable-auto-commit=true
# 提交offset延时(接收到消息后多久提交offset)
spring.kafka.consumer.auto.commit.interval.ms=1000
# 当kafka中没有初始offset或offset超出范围时将自动重置offset
# earliest:重置为分区中最小的offset;
# latest:重置为分区中最新的offset(消费分区中新产生的数据);
# none:只要有一个分区不存在已提交的offset,就抛出异常;
spring.kafka.consumer.auto-offset-reset=latest
# 消费会话超时时间(超过这个时间consumer没有发送心跳,就会触发rebalance操作)
spring.kafka.consumer.properties.session.timeout.ms=120000
# 消费请求超时时间
spring.kafka.consumer.properties.request.timeout.ms=180000
# Kafka提供的序列化和反序列化类
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
# 消费端监听的topic不存在时，项目启动会报错(关掉)
spring.kafka.listener.missing-topics-fatal=false
# 设置批量消费
# spring.kafka.listener.type=batch
# 批量消费每次最多消费多少条消息
# spring.kafka.consumer.max-poll-records=50

```

##  手动创建、修改、查询Topic
```java
------------创建topic---------------
@Configuration
public class KafkaInitialConfiguration {
 
    //创建TopicName为topic.hangge.initial的Topic并设置分区数为8以及副本数为1
    @Bean
    public NewTopic initialTopic() {
        return new NewTopic("topic.hangge.initial",8, (short) 1);
    }
}------------修改topic---------------
@Configuration
public class KafkaInitialConfiguration {
     
    //创建TopicName为topic.hangge.initial的Topic并设置分区数为10以及副本数为1
    @Bean
    public NewTopic initialTopic() {
        return new NewTopic("topic.hangge.initial",10, (short) 1 );
    }
}

------------查询topic---------------
//首先我们在配置类中注册 AdminClient 这个 Bean
@Configuration
public class KafkaInitialConfiguration {
 
    @Value("${spring.kafka.bootstrap-servers}")
    private String kafkaServers;
 
 
    @Bean
    public AdminClient adminClient() {
        Map<String, Object> props = new HashMap<>();
        //配置Kafka实例的连接地址
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaServers);
        return AdminClient.create(props);
    }
}


-----
@RestController
public class HelloController {
 
    @Autowired
    private AdminClient adminClient;
 
    @GetMapping("/hello")
    public void hello() throws ExecutionException, InterruptedException{
        DescribeTopicsResult result = adminClient.describeTopics(
                Arrays.asList("topic.hangge.initial"));
        result.all().get().forEach((k,v)->System.out.println("k: "+k+" ,v: "+v.toString()+"\n"));
    }
}

----------
AdminClient 除了查询 Topic 外，还有如下其他功能：  

-   创建 Topic：createTopics(Collection<NewTopic> newTopics)
-   删除 Topic：deleteTopics(Collection<String> topics)
-   罗列所有 Topic：listTopics()
-   查询 Topic：describeTopics(Collection<String> topicNames)
-   查询集群信息：describeCluster()
-   查询 ACL 信息：describeAcls(AclBindingFilter filter)
-   创建 ACL 信息：createAcls(Collection<AclBinding> acls)
-   删除 ACL 信息：deleteAcls(Collection<AclBindingFilter> filters)
-   查询配置信息：describeConfigs(Collection<ConfigResource> resources)
-   修改配置信息：alterConfigs(Map<ConfigResource, Config> configs)
-   修改副本的日志目录：alterReplicaLogDirs(Map<TopicPartitionReplica, String> replicaAssignment)
-   查询节点的日志目录信息：describeLogDirs(Collection<Integer> brokers)
-   查询副本的日志目录信息：describeReplicaLogDirs(Collection<TopicPartitionReplica> replicas)
-   增加分区：createPartitions(Map<String, NewPartitions> newPartitions)

  

```

## send()
```
参数说明：

-   topic：这里填写的是 Topic 的名字
-   partition：这里填写的是分区的 id，其实也是就第几个分区，id 从 0 开始。表示指定发送到该分区中
-   timestamp：时间戳，一般默认当前时间戳
-   key：消息的键
-   data：消息的数据
-   ProducerRecord：消息对应的封装类，包含上述字段
-   Message<?>：Spring 自带的 Message 封装类，包含消息及消息头

---------------------------------------------
ListenableFuture<SendResult<K, V>> send(String topic, V data);`

ListenableFuture<SendResult<K, V>> send(String topic, K key, V data);`

ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, K key, V data);`

ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, Long timestamp, K key, V data);`

ListenableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record);`

ListenableFuture<SendResult<K, V>> send(Message<?> message)`
```

## 简单生产者
```java
@RestController
public class KafkaProducer {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    // 发送消息
    @GetMapping("/kafka/normal/{message}")
    public void sendMessage1(@PathVariable("message") String normalMessage) {
        kafkaTemplate.send("topic1", normalMessage);
    }
}

```

## 带回调生产者
```java
@GetMapping("/kafka/callbackTwo/{message}")
public void sendMessage3(@PathVariable("message") String callbackMessage) {
    kafkaTemplate.send("topic1", callbackMessage).addCallback(new ListenableFutureCallback<SendResult<String, Object>>() {
        @Override
        public void onFailure(Throwable ex) {
            System.out.println("发送消息失败："+ex.getMessage());
        }
 
        @Override
        public void onSuccess(SendResult<String, Object> result) {
            System.out.println("发送消息成功：" + result.getRecordMetadata().topic() + "-"
                    + result.getRecordMetadata().partition() + "-" + result.getRecordMetadata().offset());
        }
    });
}
```

## 消费者
```java
@Component
public class KafkaConsumer {
    // 消费监听
    @KafkaListener(topics = {"topic1"})
    public void onMessage1(ConsumerRecord<?, ?> record){
        // 消费的哪个topic、partition的消息,打印出消息内容
        System.out.println("简单消费："+record.topic()+"-"+record.partition()+"-"+record.value());
    }
}

```

## 指定offset
```java
/**
 * @Title 指定topic、partition、offset消费
 * @Description 同时监听topic1和topic2，监听topic1的0号分区、topic2的 "0号和1号" 分区，指向1号分区的offset初始值为8
 **/
@KafkaListener(id = "consumer1",groupId = "felix-group",topicPartitions = {
        @TopicPartition(topic = "topic1", partitions = { "0" }),
        @TopicPartition(topic = "topic2", partitions = "0", partitionOffsets = @PartitionOffset(partition = "1", initialOffset = "8"))
})
public void onMessage2(ConsumerRecord<?, ?> record) {
    System.out.println("topic:"+record.topic()+"|partition:"+record.partition()+"|offset:"+record.offset()+"|value:"+record.value());
}

```


## 自定义分区器
```java
public class CustomizePartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // 自定义分区规则(这里假设全部发到0号分区)
        // ......
        return 0;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }
}

```


## 事务提交
```java
//1.使用 executeInTransaction 方法
@RestController
public class KafkaProducer {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
 
    // 发送消息
    @GetMapping("/test")
    public void test() {
        // 声明事务：后面报错消息不会发出去
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send("topic1","test executeInTransaction");
            throw new RuntimeException("fail");
        });
    }
}

//2. 使用 @Transactional 注解方式
//2.1 如果要使用注解方式开启事务，首先我们需要配置 KafkaTransactionManager，这个类就是 Kafka 提供给我们的事务管理类，我们需要使用生产者工厂来创建这个事务管理类
@Configuration
public class KafkaProducerConfig {
    @Value("${spring.kafka.bootstrap-servers}")
    private String servers;
    @Value("${spring.kafka.producer.retries}")
    private int retries;
    @Value("${spring.kafka.producer.acks}")
    private String acks;
    @Value("${spring.kafka.producer.batch-size}")
    private int batchSize;
    @Value("${spring.kafka.producer.properties.linger.ms}")
    private int linger;
    @Value("${spring.kafka.producer.buffer-memory}")
    private int bufferMemory;
 
 
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, servers);
        props.put(ProducerConfig.RETRIES_CONFIG, retries);
        props.put(ProducerConfig.ACKS_CONFIG, acks);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, batchSize);
        props.put(ProducerConfig.LINGER_MS_CONFIG, linger);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, bufferMemory);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return props;
    }
 
    public ProducerFactory<String, Object> producerFactory() {
        DefaultKafkaProducerFactory factory = new DefaultKafkaProducerFactory<>(producerConfigs());
        factory.transactionCapable();
        factory.setTransactionIdPrefix("tran-");
        return factory;
    }
 
    @Bean
    public KafkaTransactionManager transactionManager() {
        KafkaTransactionManager manager = new KafkaTransactionManager(producerFactory());
        return manager;
    }
}

//2.2 之后如果一个方法需要使用事务，我们只需要在该方法上添加 @Transactional 注解即可
@RestController
public class KafkaProducer {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
 
    // 发送消息
    @GetMapping("/test")
    @Transactional
    public void test() {
        kafkaTemplate.send("topic1","test executeInTransaction");
        throw new RuntimeException("fail");
    }
}

```


## 自定义过滤器
```java
@Configuration  
public class kafkaConfig {  
  
    // 监听器工厂  
    @Resource  
    private ConsumerFactory consumerFactory;  
  
    // 配置一个消息过滤策略  
    @Bean  
    public ConcurrentKafkaListenerContainerFactory myFilterContainerFactory() {  
        ConcurrentKafkaListenerContainerFactory factory =  
                new ConcurrentKafkaListenerContainerFactory();  
        factory.setConsumerFactory(consumerFactory);  
        // 被过滤的消息将被丢弃  
        factory.setAckDiscarded(true);  
        factory.setAutoStartup(true);  
        // 消息过滤策略  
        factory.setRecordFilterStrategy(new RecordFilterStrategy() {  
            @Override  
            public boolean filter(ConsumerRecord consumerRecord) {  
                // 利用时间戳将消息过滤，以下代码是五分钟  
                if (consumerRecord.timestamp() > (System.currentTimeMillis() - 5* 60 * 1000)) {  
                    return false;  
                }  
                //返回true将会被丢弃  
                return true;  
            }  
        });  
        return factory;  
    }  
}

------------------------------------------------------------------------------------------------------
@Component  
public class ConsumerForTime {  
  
  
  
    // 消息过滤监听  
    @KafkaListener(groupId = "timetest", topics = {"number"},containerFactory = "myFilterContainerFactory")  
    public void onMessage6(ConsumerRecord<?, ?> record) {  
        System.out.println(record.value());  
    }  
}
```

## 异常处理器
```java
// 新建一个[[[[异常]]]]处理器，用@Bean注入
@Configuration
public class KafkaInitialConfiguration {
@Bean
public ConsumerAwareListenerErrorHandler consumerAwareErrorHandler() {
    return (message, exception, consumer) -> {
        System.out.println("消费异常："+message.getPayload());
        return null;
    };
}

// 将这个异常处理器的BeanName放到@KafkaListener注解的errorHandler属性里面
@KafkaListener(topics = {"topic1"},errorHandler = "consumerAwareErrorHandler")
public void onMessage4(ConsumerRecord<?, ?> record) throws Exception {
    throw new Exception("简单消费-模拟异常");
}

// 批量消费也一样，异常处理器的message.getPayload()也可以拿到各条消息的信息
@KafkaListener(topics = "topic1",errorHandler="consumerAwareErrorHandler")
public void onMessage5(List<ConsumerRecord<?, ?>> records) throws Exception {
    System.out.println("批量消费一次...");
    throw new Exception("批量消费-模拟异常");
	}
}
```

## 消息转发
```java
/**
 * @Title 消息转发
 * @Description 从topic1接收到的消息经过处理后转发到topic2
 **/
@KafkaListener(topics = {"topic1"})
@SendTo("topic2")
public String onMessage7(ConsumerRecord<?, ?> record) {
    return record.value()+"-forward message";
}
```

## 启动、停止监听器
### 定时
```java
 @EnableScheduling
@Component
public class CronTimer {
    /**
     * @KafkaListener注解所标注的方法并不会在IOC容器中被注册为Bean，
     * 而是会被注册在KafkaListenerEndpointRegistry中，
     * 而KafkaListenerEndpointRegistry在SpringIOC中已经被注册为Bean
     **/
    @Autowired
    private KafkaListenerEndpointRegistry registry;

    @Autowired
    private ConsumerFactory consumerFactory;

    // 监听器容器工厂(设置禁止KafkaListener自启动)
    @Bean
    public ConcurrentKafkaListenerContainerFactory delayContainerFactory() {
        ConcurrentKafkaListenerContainerFactory container = new ConcurrentKafkaListenerContainerFactory();
        container.setConsumerFactory(consumerFactory);
        //禁止KafkaListener自启动
        container.setAutoStartup(false);
        return container;
    }

    // 监听器
    @KafkaListener(id="timingConsumer",topics = "topic1",containerFactory = "delayContainerFactory")
    public void onMessage1(ConsumerRecord<?, ?> record){
        System.out.println("消费成功："+record.topic()+"-"+record.partition()+"-"+record.value());
    }

    // 定时启动监听器
    @Scheduled(cron = "0 42 11 * * ? ")
    public void startListener() {
        System.out.println("启动监听器...");
        // "timingConsumer"是@KafkaListener注解后面设置的监听器ID,标识这个监听器
        if (!registry.getListenerContainer("timingConsumer").isRunning()) {
            registry.getListenerContainer("timingConsumer").start();
        }
        //registry.getListenerContainer("timingConsumer").resume();
    }

    // 定时停止监听器
    @Scheduled(cron = "0 45 11 * * ? ")
    public void shutDownListener() {
        System.out.println("关闭监听器...");
        registry.getListenerContainer("timingConsumer").pause();
    }
}

```

### 动态
```java
//1. 消费者
@Component
public class KafkaConsumer {
    // 消费监听
    @KafkaListener(id = "myListener1", topics = {"topic1"})
    public void listen1(String data) {
        System.out.println("监听器收到消息：" + data);
    }
}
/**
KafkaListenerEndpointRegistry 在 SpringIO 中已经被注册为 Bean，直接注入使用即可。
还需要注意一下启动监听容器的方法，resume 是恢复的意思不是启动的意思。所以我们需要判断容器是否运行，如果运行则调用 resume 方法，否则调用 start 方法
*/
//2. 定义两个 controller 接口分别通过 KafkaListenerEndpointRegistry 来控制监听器的开启、关闭

@RestController
public class KafkaProducer {
 
    @Autowired
    private KafkaListenerEndpointRegistry registry;
 
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
 
    // 发送消息
    @GetMapping("/test")
    public void test() {
        System.out.println("监听器发送消息!");
        kafkaTemplate.send("topic1", "1条测试消息");
    }
 
    // 开启监听
    @GetMapping("/start")
    public void start() {
        System.out.println("开启监听");
        //判断监听容器是否启动，未启动则将其启动
        if (!registry.getListenerContainer("myListener1").isRunning()) {
            registry.getListenerContainer("myListener1").start();
        }
        registry.getListenerContainer("myListener1").resume();
    }
 
    // 关闭监听
    @GetMapping("/stop")
    public void stop() {
        System.out.println("关闭监听");
        //判断监听容器是否启动，未启动则将其启动
        registry.getListenerContainer("myListener1").pause();
    }
}

//3.1 禁止监听器自启动
@Configuration
public class KafkaInitialConfiguration {
 
    // 监听器工厂
    @Autowired
    private ConsumerFactory consumerFactory;
 
    // 监听器容器工厂(设置禁止KafkaListener自启动)
    @Bean
    public ConcurrentKafkaListenerContainerFactory delayContainerFactory() {
        ConcurrentKafkaListenerContainerFactory container =
                new ConcurrentKafkaListenerContainerFactory();
        container.setConsumerFactory(consumerFactory);
        //禁止自动启动
        container.setAutoStartup(false);
        return container;
    }
}
//3.2 然后将这个容器工厂的 BeanName 放到 @KafkaListener 注解的 containerFactory 属性里面。这样该监听器就不会自动启动了
@Component
public class KafkaConsumer {
    // 消费监听
    @KafkaListener(id = "myListener1", topics = {"topic1"},
            containerFactory = "delayContainerFactory")
    public void listen1(String data) {
        System.out.println("监听器收到消息：" + data);
    }
}
```

## 并发、批量消费
### 批量消费
```java
----application.properties---


# 设置批量消费
spring.kafka.listener.type=batch
# 批量消费每次最多消费多少条消息
spring.kafka.consumer.max-poll-records=5

-------------
@Component
public class KafkaConsumer {
    // 消费监听
    @KafkaListener(topics = {"topic3"})
    public void listen1(List<ConsumerRecord<String, Object>> records) {
        System.out.println("收到"+ records.size() + "条消息：");
        System.out.println(records);
    }
}

```
### 并发消费
```java
----application.properties---
# 并发数设为3  
spring.kafka.listener.concurrency=3
----------------------------
//配置完毕后，消费者监听这边不需要修改
```


# 6 Windows环境下的Kafka安装
## 6.1 zookeeper的安装
## 6.2 Kafka的安装

# 7 Windows kafka常用指令




# 参考
https://www.hangge.com/blog/cache/detail_2957.html
https://blog.csdn.net/qq_48496502/article/details/125316086
![[01_尚硅谷大数据技术之Kafka 1.pdf]]


#kafka增加副本
```
bin/kafka-topics.sh --bootstrap-server 49.232.114.55:9092 --describe --topic AIS-hanzhen

vim increase-replication-factor-AIS-hanzhen.json

{
"version":1,
"partitions":[{"topic":"AIS-hanzhen","partition":0,"replicas":[0,1]},
{"topic":"AIS-hanzhen","partition":1,"replicas":[1,2]},
{"topic":"AIS-hanzhen","partition":2,"replicas":[2,1]}]
}

执行
bin/kafka-reassign-partitions.sh --bootstrap-server 49.232.114.55:9092 --reassignment-json-file increase-replication-factor-AIS-hanzhen.json --execute

验证
bin/kafka-reassign-partitions.sh --bootstrap-server 49.232.114.55:9092 --reassignment-json-file increase-replication-factor-AIS-hanzhen.json --verify
```

#删除topic
```
bin/kafka-topics.sh --bootstrap-server 49.232.114.55:9092 --delete --topic test
```

#Kafka启动与关闭
```
开启：
bin/kafka-server-start.sh -daemon config/server.properties

关闭：
bin/kafka-server-stop.sh
```

#消费消息
```
bin/kafka-console-consumer.sh --bootstrap-server 192.168.0.235:9092 --topic test
```