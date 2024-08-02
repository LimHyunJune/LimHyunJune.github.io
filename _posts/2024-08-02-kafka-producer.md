---
layout: post
title: Apache Kafka Producer
subtitle:
excerpt_image: 
author: Hyunjune
categories: kafka
tags: producer
---
{% raw %}
### Producer 프로세스
![img.png](https://limhyunjune.github.io/assets/images/producer.png)

- Broker 전송 성공 여부에 따른 재전송은 동기식에서만 동작
- 실패 시 error 반환
```java
kafkaProducer.send().get()
```


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


{% endraw %}
