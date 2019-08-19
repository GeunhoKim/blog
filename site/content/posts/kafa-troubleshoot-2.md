---
author: "Kim, Geunho"
date: 2016-06-01
title: "Kafka - org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = NoNode for $BROKER_PATH"
---


# 발생 원인 및 해결 방법
Kafka consumer가 zookeeper 접속에 실패하면 발생하는 에러이다. 그 원인에는 여러가지 이유가 있는데 그 중 timeout 시간이 짧아 발생한 상황에 대해 정리한다.  

Kafka는 consumer group에 변화가 생기면 partition을 다시 분배하기 위해 rebalance가 일어난다. 그런데 Broker가 Zookeeper로 heartbeat를 전송하다가 실패하면 그 broker가 죽은 것으로 인지하여 rebalance가 발생하기도 한다.  

heartbeat의 timeout 시간을 조정하는 설정값이 바로 `zookeeper.session.timeout.ms`[^1]인데 기본 값이 6000ms으로 짧기 때문에, Kafka의 부하가 큰 경우 timeout이 여러번 발생하게 되면서 rebalance가 계속해서 일어날 수 있다. 이때 Kafa의 rebalance 작업 때문에 consumer는 아무런 일도 할 수 없게 되는 것이다.  

따라서 이러한 경우에 다음과 같이 Kafka의 기본 값을 조정하면 문제가 해결될 수 있다.

```bash
# 혹은 그 이상으로 설정, 대신 실제로 broker가 죽은 경우, 그만큼 판단이 느려질 수 있음
zookeeper.session.timeout.ms = 15000 

# rebalance를 재시작하기까지 기다리는 시간. 기본 값은 2000ms로 짧음
rebalance.backoff.ms = 10000 
```

[^1]: 전체 설정값에 대한 정보는 https://kafka.apache.org/082/documentation.html#consumerconfigs 참고


