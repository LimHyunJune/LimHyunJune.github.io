---
layout: post
title: Apache Kafka Consumer
subtitle:
excerpt_image: https://limhyunjune.github.io/assets/images/consumer.png
author: Hyunjune
categories: kafka
tags: [consumer, rebalancing, group coordinator, heartbeat]
---
{% raw %}
### Consumer 개요
- 브로커의 topic 메시지 읽는 역할 수행
- 모든 consumer 들은 고유한 group id를 가지는 consumer group에 속해야 함
- Fetcher, ConsumerClientNetwork 등의 주요 내부 객체와 별도의 HeartBeat Thread를 생성


![img.png](https://limhyunjune.github.io/assets/images/consumer.png)

- partition은 consumer group에서 단 하나의 consumer에만 할당됨
- 동일 consumer group 내 consumer들은 작업량을 최대한 균등하게 분배
- 서로 다른 consumer group의 consumer들은 분리되어 독립적으로 동작

```
kafka-console-consumer --bootstrap-server localhost:9092 --group 그룹명 --topic 토픽명
--property pring.key=true --property print.partition=true
```

#### simple consumer
```java
String topic = "simple-topic";
Properties props = new Properties();
props.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.56.101:9092");
props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "group-01");

KafkaConsumer<String,String> kafkaConsumer = new KafkaConsumer<>(props);
kafkaConsumer.subscribe(List.of(topic));
while(true)
{
 ConsumerRecords<String,String> consumerRecords = kafkaConsumer.poll(Duration.ofMillis(1000));
 for(ConsumerRecord record : consumerRecords)
 {
     logger.info("key : {} value : {} partitions : {}". record.key(), record.value(), record.partition());
 }
}
kafkaConsumer.close();
```
- subscribe에 topic은 collection 또는 pattern으로 지정 가능
- `kafkaConsumer.poll(Duration.ofMillis(1000))` 
  - 가져올 데이터가 하나도 없는 경우 1000ms 만큼 기다린 후 return

![img.png](https://limhyunjune.github.io/assets/images/fetcher.png)
- Fetcher는 Linked Queue에 데이터가 없는 경우 ConsumerClient Network에 데이터를 가져올 것을 요청
- Linked Queue에 데이터가 있는 경우 Fetcher는 데이터 가져오고 poll() 수행 완료

<br>
<hr>

### Consumer Fetcher 관련 주요 설정 파라미터

- fetch.min.bytes
  - fetcher가 record를 읽어들이는 최소 bytes (default = 1)
  - fetcher가 ConsumerClient Network를 통해 record를 읽어오도록 했을 때 지정한 bytes만큼 쌓이지 않으면 broker는 전송하지 않음
- fetch.max.wait.ms
  - 브로커에 fetch.min.bytes 이상의 메시지가 쌓일 때까지 최대 대기 시간 (default = 500ms)
  - 이 시간까지 쌓이지 않으면 그냥 가져감
- fetch.max.bytes
  - fetcher가 브로커에서 한 번에 가져올 수 있는 최대 데이터 bytes (default 50MB)
- max.partition.fetch.bytes
  - fetcher가 브로커에서 파티션 별 한 번에 가져올 수 있는 bytes
- max.poll.records
  - fetcher가 브로커에서 한 번에 가져올 수 있는 레코드 수 (default 500)
  - fetch.max.bytes 초과하지 않는 범위내에서 레코드 수 제한함


만약 가져올 데이터가 많은 경우 max.partition.fetch.byte 크기를 조절해야 하고, 적다면 fetch.min.bytes 크기를 조절해야 함
- 가장 최신의 offset 데이터를 가져오고 있다면 fetch.min.bytes 만큼 가져오고 있거나 데이터가 min만큼 차지 않아서 fetch.min.bytes만큼 기다렸다가 return 한 경우일 수 있음
- 오랜 과거 offset 데이터를 가져온 다면 최대 max.partition.fetch.bytes만큼 파티션에서 읽은 뒤 반환

<br>
<hr>


### Consumer의 auto.offset.reset

- `__consumer_offsets`에 consumer group이 offset 정보를 가지고 있지 않을 시 consumer가 접속 시 파티션의 처음 offset부터 가져올 것인지, 마지막 offset 이후부터 가져올 것인지를 설정
  - 만약 가지고 있으면 기록된 offset을 읽음
- offsets.retention.minutes
  - __consumer_offsets에 유지되는 offset 기간 (default 7일)
  - consumer가 종료되어도 offset은 유지됨
- 해당 topic이 삭제되고 재생성되는 경우는 topic에 대한 consumer group의 offset 정보는 0으로 기록됨

<br>
<hr>

### Group Coordinator와 Consumer Group
- consumer group 내 새로운 consumer가 추가되거나 기존 consumer가 종료될 때, 또는 topic에 새로운 partition이 추가될 때 broker의 group coordinator는 consumer group내의 consumer 들에게 파티션을 재할당하는 rebalancing 수행 지시
- consumer group은 group coordinator를 통해 __consumer_offsets 토픽에 그룹 내 consumer들의 offset을 기록하는데, __consumer_offsets에는 50개의 파티션이 존재, consumer group 별 작성할 파티션이 존재하는 브로커가 group coordinator가 됨

![img.png](https://limhyunjune.github.io/assets/images/groupcoordinator.png)

#### group coordinator
- consumer들의 join group 정보
- partition 매핑 정보
- consumer의 heartbeat 관리

1) consumer group 내 consumer가 broker에 최초 접속 요청 시 group coordinator가 생성
2) 동일 group.id로 여러 개의 consumer가 broker의 group coordinator로 접속
3) 가장 빨리 group에 join 요청 한 consumer에게 leader consumer로 지정
4) leader는 파티션 할당 전략에 따라 consumer들에게 파티션 할당
5) leader는 최종 할당 된 파티션 정보를 group coordinator에게 전달
6) 정보 전달 성공을 공유한 뒤 개별 consumer들은 할당된 파티션에서 메시지 읽음



{% endraw %}
