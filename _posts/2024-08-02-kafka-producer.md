---
layout: post
title: Apache Kafka Producer
subtitle:
excerpt_image: https://limhyunjune.github.io/assets/images/producer.png
author: Hyunjune
categories: kafka
tags: producer
---
{% raw %}
### Producer 개요
- Producer는 Topic에 메시지를 보냄(메시지 write)
- Producer는 성능/로드밸런싱/가용성/업무 정합성등을 고려하여 어떤 브로커의 파티션으로 메시지를 보내야 할지 전략적으로 결정됨

#### Producer Record
![img.png](https://limhyunjune.github.io/assets/images/producerrecord.png)



#### simple producer
```java
String topicName = "simgple-producer";
Properties props = new Properties();
props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.56.101:9092");
props.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

KafkaProducer<String,String> kafkaProducer = new KafkaProducer<String,String>(props);
ProducerRecord<String,String> producerRecord = new Producer<>(topicName, "hello");

kafkaProducer.send(producerRecord);
kafkaProducer.flush();
kafkaProducer.close();
```
- kafka에서 제공하는 serializer
  - StringSerializer
  - ShortSerializer
  - IntegerSerializer
  - DoubleSerializer
  - LongSerializer
  - BytesSerializer

<br>
<hr>

## Producer 메시지 파티셔닝

### key 값을 가지지 않는 메시지 전송
- 메시지는 producer를 통해 전송 시 partitioner를 통해 어떤 파티셔너로 전송되어야 하는지 미리 결정됨
- key 값을 가지지 않는 경우 Round Robin, Sticky Partitioning 등의 전략이 선택되어 파티션 별로 메시지가 전송됨
- Topic이 여러 개의 파티션을 가질 때 `전송 순서가 보장되지 않은 채`로 consumer에서 읽혀질 수 있음


#### 라운드 로빈
- kafka 2.4 이전 default 파티셔닝 전략
- 최대한 메시지를 파티션에 균등하게 분배하려는 전략으로써 메시지 배치를 순차적으로 다른 파티션으로 전송
- 메시지가 배치 데이터를 빨리 채우지 못하면서 전송이 늦어지거나 배치를 다 채우지 못하고 전송하면서 성능 이슈

![img.png](https://limhyunjune.github.io/assets/images/roundrobin.png)

#### 스티키 파티셔닝
- kafka 2.4 이후 default 파티셔닝 전략
- 라운드 로빈의 성능을 개선하고자 특정 파티션으로 전송되는 하나의 배치에 메시지를 빠르게 먼저 채워서 보내는 방식

![img.png](https://limhyunjune.github.io/assets/images/sticky.png)


### key 값을 가지는 메시지 전송
- 특정 key 값을 가지는 메시지는 `특정 파티션으로 고정 전송됨`
- 특정 key 값을 가지는 메시지는 단일 파티션 내에서 전송 순서가 보장되어 consumer에서 읽힘
  - 하나의 파티션에서만 메시지 순서 보장

![img.png](https://limhyunjune.github.io/assets/images/producerkey.png)

#### key 메시지 전송
- 전송
```
kafka-console-producer --bootstrap-server localhost:9092 --topic 토픽명 --property key.separator=: --property parse.key=true
```
- 수신
```
kafka-console-consumer --bootstrap-server localhost:9092 --topic 토픽명 --property print.key=true --from-beginning
```

### DefaultPartitioner
- kafka는 기본적으로 DefaultPartitioner를 통해 partition 지정
- key를 가지는 메시지의 경우 key 값을 해싱하여 파티션별로 균일하게 전송

```java
public int partition(String topic, Object key, bytes[] keyBytes, Bytes[] valueBytes, Cluster cluster, int numPartition)
{
    if(keyBytes == null){
        return StickyPartitionCache.partition(topic,cluster);    
    }
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartition;
}
```

### Custom Partitioner 구현
- Partitioner interface 구현, partition() 메소드 구현 필요
```java
public class CustomPartitioner implements Partitioner {
  public static final Logger logger = LoggerFactory.getLogger(CustomPartitioner.class.getName());
  private final StickyPartitionCache stickyPartitionCache = new StickyPartitionCache();
  private String specialKeyName;

  @Override
  public void configure(Map<String, ?> configs) {
    specialKeyName = configs.get("custom.specialKey").toString();
  }

  @Override
  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitionInfoList = cluster.partitionsForTopic(topic);
    int numPartitions = partitionInfoList.size();
    int numSpecialPartitions = (int)(numPartitions * 0.5);
    int partitionIndex = 0;

    if (keyBytes == null) {
      //return stickyPartitionCache.partition(topic, cluster);
      throw new InvalidRecordException("key should not be null");
    }

    if (((String)key).equals(specialKeyName)) {
      partitionIndex = Utils.toPositive(Utils.murmur2(valueBytes)) % numSpecialPartitions;
    }
    else {
      partitionIndex = Utils.toPositive(Utils.murmur2(keyBytes)) % (numPartitions - numSpecialPartitions) + numSpecialPartitions;
    }
    logger.info("key:{} is sent to partition:{}", key.toString(), partitionIndex);

    return partitionIndex;
  }

  @Override
  public void close() {

  }
}
```
- `configs.get("custom.specialKey").toString();` 
  - producer에 등록한 config 값을 가져올 수 있음
<br>
<hr>

### send() 메소드 호출 프로세스

![img.png](https://limhyunjune.github.io/assets/images/producer.png)
- Kafka Producer 전송은 Producer Client의 별도 Thread가 전송을 담당한다는 점에서 기본적으로 Thread간 Async 전송
- Main Thread가 send( ) 메소드를 호출하여 메시지 전송을 시작하지만 바로 전송되지 않으며 내부 Buffer에서 토픽 파티션에 따라 Record Batch 단위로 묶인 뒤 전송됨
- producer client의 내부 메모리 (**Record Accumulator**)에 여러 개의 batch들로 buffer.memory 설정 사이즈만큼 보관 가능
- 복수 개의 batch 한꺼번에 전송 가능
- ``batch.size``만큼 레코드가 쌓이면 sender thread를 통해 전송
- ``linger.ms``를 줄이면 batch 사이즈를 다 채우기 전에 전송할 수 있도록 할 수 있음 (20ms 이하 권장)

#### max.inflight.requests.per.connection
- 한 번에 전송 가능한 메시지 배치 개수 (default=5)
- 1보다 큰 경우 일부 배치의 전송 실패 시 순서 뒤집힐 수 있음

<br>
<hr>

### 메시지 동기식/비동기식 전송
#### Sync 방식
- Producer는 broker로부터 해당 메시지를 성공적으로 받았다는 Ack 메시지를 받은 후 메시지 전송
- 동기 처리시 1개씩 전송하며 ack를 받아야 하므로 batch 처리 불가능, 전송은 batch 레벨이지만 메시지는 1개

동기화 코드
```java
Future<RecordMetadata> future = producer.send();
RecordMetadata metadata = future.get();

# inline
RecordMetadata metadata = producer.send().get();
```

#### Async 방식
- Callback을 이용한 비동기 메시지 전송 가능
  - 비동기적으로 메시지를 보내면서 RecordMetadata를 client가 받을 수 있는 방식 제공
- Async 전송 시 batch를 통해 전송함
```java
producer.send(producerRecord, new Callback() {
  @Override
  public void onCompletion( RecordMetadata metadata, Exception exception ) {
    if (exception == null) {
      System.out.println("received metadata \n" +
        "Topic:" + metadata.topic() + "\n" +
        "Partition:" + metadata.partition() + "\n" +
        "Offset:" + metadata.offset());
    } else {
        exception.printStackTrace();
    }
  }
```

#### Custom Callback 생성
- Callback 인터페이스 상속
```java
public class CustomCallback implements Callback{
  @Override
  public void onCompletion( RecordMetadata metadata, Exception exception ) {
      if (exception == null) {
          System.out.println("received metadata \n" +
                  "Topic:" + metadata.topic() + "\n" +
                  "Partition:" + metadata.partition() + "\n" +
                  "Offset:" + metadata.offset());
      } else {
          exception.printStackTrace();
      }
   }
}

```

<br>
<hr>


{% endraw %}
