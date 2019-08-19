---
author: "Kim, Geunho"
date: 2016-05-09
title: "kafka - kafka.common.InconsistentBrokerIdException: Configured brokerId $id doesn’t match stored brokerId $id in meta.properties"
---


# 발생 원인 및 해결 방법
Kafka broker 서버의 설정 파일 (`conf/server.properties`) 에 명시된 `broker.id` 값과 로그 파일이 저장되는 폴더 (`kafka-logs/meta.properties`) 에 명시된 `broker.id` 값이 서로 달라서 발생하는 문제이다.  

Kafka 클러스터를 처음 구성할 때, 설정 파일 (e.g. zookeeper 연결 문자열) 수정시 brokerid를 자동으로 생성, 할당하는 과정에서 불일치가 발생할 수 있다. 아래 방법들 중 하나로 불일치를 해결하면 된다.

* 로그 파일이 저장된 폴더의 `meta.properties` 파일에 `broker.id` 값을 `server.properties`에 명시된 값으로 수정 후 broker 재시작
* 아직 운영하기 전인 클러스터라면, 로그 파일이 저장된 폴더를 모두 삭제 후 broker 재시작  

