---
layout: post
title: Kafka Connect 개요
subtitle:
excerpt_image: https://limhyunjune.github.io//assets/images/connect-cluster.png
author: Hyunjune
categories: kafka
tags: [connect, spooldir source connector]
---
{% raw %}
### Kafka Connect 개요
- Kafka 메시지 시스템 (Broker/Producer/Consumer)를 기반으로 다양한 데이터 소스 시스템 (ex: RDBMS)에서 발생한 데이터 이벤트를 다른 데이터 타겟 시스템으로
- 별도의 client 코딩 없이 seamless하게 실시간 전달하기 위해 만들어진 kafka component'

#### Kafka Connect 활용
- Data Pipeline
- DW ETL 활용
- Micro Service
- DB offloading

#### Kafka Connect 주요 구성요소
- Connector
  - JDBC Source / Sink
  - Debezium CDC Source
  - Elasticsearch Sink
  - File Connector
  - MongoDB Source / Sink
- Transformation
  - SMT
- Converter
  - JsonConverter
  - AvroConverter

![img.png](https://limhyunjune.github.io/assets/images/connector.png)

<br>
<hr>

### Connect Cluster 아키텍처
![img.png](https://limhyunjune.github.io/assets/images/connect-cluster.png)
- connect cluster는 동일 group id를 지정하는 것으로 클러스터를 구축하고 kafka cluster 내의 internal topic들을 통해 서로의 상태 정보를 공유함

#### Connect, Connector, Worker, Task 정의
- connect는 connector를 기동시키기 위한 framework를 갖춘 JVM process 모델
- connect process를 `worker process`로 지칭함
- connect는 서로 다른 여러 개의 connector instance를 자신의 framework 내부로 로딩하고 호출 및 수행 가능
- connector instance 실제 수행은 thread 레벨로 수행되며 이를 `task`라고 한다
- connector가 병렬 thread에서 수행이 가능한 경우 여러 개의 task thread들로 해당 connector 수행 가능
- connect는 connect cluster로 구성될 수 있으며, 1개의 노드에서 여러 개의 worker process 또는 여러 개의 노드에서 여러 개의 worker process 들로 connect cluster 구성 가능
- connect 유형은 `standalone`과 `distributed mode`로 나뉨
  - standalone : 단일 worker cluster만 가능
  - distributed mode : 여러 worker로 가능

<br>
<hr>

### Connector 설치 
- connect 설정의 `plugin.path`에 connector 설치
- confluent-hub 사용

#### Spool Source Connector
- 특정 디렉토리에 위치한 csv, json 포맷 등의 파일들을 event message로 만들어서 kafka로 전송하는 source connector
- 해당 디렉토리를 주기적으로 모니터링 수행하면서 새로운 파일이 생성될 때마다 kafka로 메시지 전송

#### Kafka Connect에 새로운 Connector 생성 순서
1. Connector를 다운로드 받음. Connector는 여러 개의 jaf file로 구성 <br>
2. Connector를 plugin.path로 지정된 디렉토리에 별도의 서브 디렉토리를 만들어 jar file을 이동시켜야함 <br>
3. Connect는 기동 시에 plugin.path로 지정된 디렉토리 밑의 서브 디렉토리들에 위치한 모든 jar 파일을 로딩함. 신규로 connector를 connect로 올릴 시에는 반드시 connect를 재기동해야 반영됨 <br>
4. Connect는 Connector 명, Connector 클래스 명, Connector 고유 환경 설정 등을 REST API를 통해 Connect에 전달하여 새롭게 생성
5. REST API 성공 response와 별개로 connect log 메시지를 반드시 확인해야 함

#### Connector 로딩 확인
```
curl -X GET -H "Content-Type : application/json" http://localhost:8083/connector-plugins
```

{% endraw %}
