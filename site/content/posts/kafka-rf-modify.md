---
author: "Kim, Geunho"
date: 2020-06-01
title: "Kafka - Replication Factor 변경하기"
---

Kafka topic 파티션의 Replication Factor(RF)는 broker 설정 중 `offsets.topic.replication.factor`에 의해 결정된다.
기본 값은 3으로, 하나의 파티션이 총 세 개로 분산 저장되는 것이다.

Kafka 바이너리에 포함된 `kafka-reassign-partitions.sh` 도구를 이용하면 topic의 RF를 변경할 수 있다. 

# 사용방법
먼저 변경하려는 topic의 상태를 확인해보자.
```bash
# zookeeper1 서버에서 my-topic 상태를 출력
bin/kafka-topics.sh --zookeeper zookeeper1:2181 --topic my-topic --describe
Topic:my-topic   PartitionCount:2    ReplicationFactor:2 Configs:
  Topic: my-topic    Partition: 0    Leader: 5   Replicas: 5,7 Isr: 5,7
  Topic: my-topic    Partition: 1    Leader: 7   Replicas: 7,5 Isr: 5,7
```
두 개의 파티션(0,1)과 RF는 2임을 알 수 있다. 그리고 파티션은 브로커(5,7)에 있다.
브로커는 5, 6, 7번 세 개로 구성되어 있다고 가정하자.
```
  my-topic
  +---------------+ +----------------+ +----------------+
  | 5             | | 6              | | 7              |
  | +----+ +----+ | |                | | +----+ +----+  |
  | |0   | |1'  | | |                | | |0'  | |1   |  |
  | |    | |    | | |                | | |    | |    |  |
  | +----+ +----+ | |                | | +----+ +----+  |
  +---------------+ +----------------+ +----------------+

```
`0'` `1'`는 복제본을 나타낸다.
여기서 RF를 3으로 늘려서 복제본을 6번 브로커에도 갖도록 하려면 어떻게 해야 할까?

먼저 구성하려는 파티션 구성을 json 형식으로 다음과 같이 저장한다.
```bash
touch rf.json
vim rf.json

{
    "version":1,
    "partitions":[
        {
            "topic":"my-topic",
            "partition":0,
            "replicas":[5,7,6]
        },
        {
            "topic":"my-topic",
            "partition":1,
            "replicas":[7,5,6]
        }
    ]
} 
```
`replicas`의 첫 번째 원소가 해당 파티션의 리더 브로커가 되므로 파티션별 주의해서 순서를 지정하도록 하자.

이제 생성한 파일과 리파티셔닝 도구를 이용해서 RF를 늘려준다. 
```bash
bin/kafka-reassign-partitions.sh \
    --zookeeper zookeeper1:2181 \
    --reassignment-json-file rf.json --execute
Current partition replica assignment
```

다시 topic 상태를 확인해보자.
```bash
# zookeeper1 서버에서 my-topic 상태를 출력
bin/kafka-topics.sh --zookeeper zookeeper1:2181 --topic my-topic --describe
Topic:my-topic   PartitionCount:2    ReplicationFactor:3 Configs:
  Topic: my-topic    Partition: 0    Leader: 5   Replicas: 5,7,6 Isr: 5,7,6
  Topic: my-topic    Partition: 1    Leader: 7   Replicas: 7,5,6 Isr: 7.5,6
```
```
  my-topic
  +---------------+ +----------------+ +----------------+
  | 5             | | 6              | | 7              |
  | +----+ +----+ | | +----+ +----+  | | +----+ +----+  |
  | |0   | |1'  | | | |0'  | |1'  |  | | |0'  | |1   |  |
  | |    | |    | | | |    | |    |  | | |    | |    |  |
  | +----+ +----+ | | +----+ +----+  | | +----+ +----+  |
  +---------------+ +----------------+ +----------------+

```
이제 6번 브로커에도 0, 1 파티션의 복제본이 생성되었고, RF는 3이 되었다.
