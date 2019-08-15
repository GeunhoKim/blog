---
author: "Kim, Geunho"
date: 2019-08-05
draft: true
title: "Hadoop YARN"
---


# Hadoop 1
YARN에 대해서 얘기하기 전에, 먼저 하둡 1에서 Job의 실행과 관리가 어떻게 이루어졌는지 살펴본다.  
[NameNode](/posts/hadoop-namenode/)와 [DataNode](/posts/hadoop-datanode/)가 하둡의 분산 파일 시스템인 HDFS를 구성하는 주요 컴포넌트라면, MapReduce job을 구동하기 위해 JobTracker와 TaskTracker 컴포넌트가 필요하다.  

MapReduce job은 여러 개의 mapper과 reducer task로 이루어지는데, JobTracker는 HDFS의 NameNode처럼 단일 마스터 노드로서 job의 mapper, reducer task를 스케쥴링하고 관제한다. 만약 실패한 task가 있다면 다른 TaskTracker로 task를 재할당한다. TaskTracker는 task를 실행한다.

![JobTracker & TaskTracker](/hadoop-yarn-1.png) _그림 1. JobTracker와 TaskTracker의 Master/Slave 구조_

그림 1에서와 같이, TaskTracker는 DataNode가 동작하는 노드에 함께 구동되고 JobTracker는 NameNode와는 별도의 노드에서 동작한다.  
JobTracker도 단일 마스터 노드이기 때문에 NameNode처럼 SPOF[^1] 문제가 있는데 장애가 발생하면 MapReduce job이 모두 실패하고 복구 되기 전까지는 다시 실행할 수 없다. JobTracker는 별도의 컴포넌트로 동작하지만 HDFS 위에서 동작하도록 설계되었다. 하지만 JobTracker와 TaskTracker의 장애는 HDFS 동작 환경에는 영향을 주지 않는다.  

이러한 특성과 JobTracker가 단일 마스터 노드라는 점의 한계가 만나면서, 필요할 때에만 JobTracker와 데이터 사이즈에 알맞는 TaskTracker를 띄워 MapReduce 클러스터를 구성해서 job을 실행하고, 끝나면 다시 종료하는 Hadoop on Demand(HoD) 기술이 발전했다.[^2] 이 기술은 Hadoop 2에서 YARN의 설계에 직접적인 영향을 미쳤다.  

<br />
# Architecture
Hadoop 2가 되면서 JobTracker와 TaskTracker는 더 이상 사용되지 않는다. 대신 이를 완전히 대체하는 YARN(Yet Another Resource Negotiator)이 등장했다. YARN은 역할에 따라 컴포넌트를 세분화했다.

* ResourceManager  
  * ApplicationsManager  
  * Scheduler  
* NodeManager  
* ApplicationMaster  
* Container  

마스터 노드인 ResourceManager는 ApplicationsManager와 Scheduler라는 두 개의 주요 컴포넌트로 이루어져 있고, NodeManager는 DataNode가 동작하는 워커 노드에서 동작한다.  

![YARN Architecture](/hadoop-yarn-2.png) _그림 2. YARN의 구조_

그림 2를 살펴보면, JobTracker의 역할을 ResourceManager가 YARN의 마스터 노드로서 가져갔고, NodeManager는 TaskTracker의 역할을 가져가면서 각 워커 노드에서 동작하는 구조를 갖게 되었다.  
마스터와 워커 노드 모두 기능에 따라 여러 프로세스로 나뉘어졌는데, ResourceManager는 클러스터 전체 리소스 내에서 다양한 종류의 애플리케이션이 동작할 수 있도록 총괄하는 역할을 담당하며, 각 스케쥴링과 애플리케이션 실행은 마스터 노드 내에서 함께 동작하는 Scheduler와 ApplicationsManager가 맡는다.  

NodeManager은 마치 DataNode가 NameNode에게 heartbeat을 주기적으로 보내고 블록 리포트를 보내는것 처럼, 각 노드의 리소스를 관제하면서 ResourceManager에게 주기적으로 heartbeat와 리소스 관제 내용을 ResourceManage에게 보낸다.  

YARN은 이렇듯 리소스 관리와 애플리케이션 스케쥴링과 모니터링 기능을 서로 다른 컴포넌트에서 동작하도록 역할을 분리했다. 각 측면에 대해서 더 상세히 살펴보도록 한다.  

# Resource management

# Application scheduling and monitoring

# HA

[^1]: 단일 지점 실패, Single point of failure
[^2]: 뿐만 아니라 HDFS 클러스터도 필요한 만큼 여러 개를 구성하거나, 새로운 버전을 띄워 검증하는 용도로도 쓰였다.