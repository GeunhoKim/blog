---
author: "Kim, Geunho"
date: 2019-07-23T20:21:09+09:00
draft: false
title: "Hadoop NameNode"
---


## Architecture
하둡 클러스터는 단일 마스터 노드와 여러 대의 워커 노드로 이루어진 Master/Slave 구조이다. 마스터 노드는 NameNode라 불리며 하둡 분산 파일 시스템(HDFS, Hadoop Distributed File System)의 네임스페이스를 관리하며, 워커 노드인 DataNode는 파일 시스템의 파일 블록을 저장하는 역할을 한다.

![하둡 클러스터 구조](/hadoop-namenode-1.png) _그림 1. Master/Slave 구조_
 
HDFS는 MapReduce 프레임워크와 함께 하둡을 구성하는 한 줄기인데, 용량이 큰 규모의 파일을 분산 저장하기 위해 설계되었다. 큰 용량의 파일이란 수십 기가바이트에서 테라바이트까지의 규모를 뜻하는데, 이러한 파일을 블록 단위로 나누어 저장한다. 

NameNode는 분산 파일 시스템의 네임스페이스를 관리한다. 파일 시스템 트리와 트리 안의 모든 파일 및 디렉토리에 대한 메타 정보를 유지하고 있다. 파일이 어느 노드에 위치한 블록으로 구성되어 있는지 알고 있으며, 모든 변경 사항을 기록한다. 

이와 같은 정보는 NameNode의 호스트 OS의 로컬 파일 시스템에 두 가지 종류의 파일로 지속해서 기록한다. 파일의 변경 사항은 트랜잭션 로그에 해당하는 `EditLog`에 저장되고 전체 이미지 정보는 `FsImage`에 저장된다. 이 파일들은 클러스터 장애로 인해 NameNode를 재시작할 때 파일 시스템의 네임스페이스와 메타 정보를 복구하는데 사용된다. 

* EditLog : HDFS의 모든 변경 사항에 대한 트랜잭션 로그 정보  
* FsImage : HDFS의 전체 이미지 정보  
* 두 정보는 NameNode가 동작하는 OS의 로컬 파일 시스템에 기록되고 NameNode 복구시 사용

![네임스페이스와 메타 정보](/hadoop-namenode-2.png) _그림 2. EditLog와 FsImage. 클라이언트가 NameNode로 명령을 내리면 NameNode는 파일 변경 사항을 EditLog 파일에 기록한다. NameNode가 재시작 되면 네임스페이스를 복구하기 위해 FsImage를 먼저 읽어들인 후 EditLog의 내용을 차례로 병합하여 기동한다. EditLog 수가 늘어날 수 록 기동 시간은 늘어나게 된다._

---


## Persistency
만약 NameNode 장비에 이상이 생겨서 프로세스가 다운된다면 어떻게 될까? 단일 마스터 노드인 NameNode가 중지되는 즉시 HDFS는 사용할 수 없는 상태가 된다. 특히 NameNode에 디스크 장애가 발생한다면, 로컬 파일 시스템에 저장되어 있는 FsImage와 EditLog 모두 유실될 수 있는데 초창기의 하둡 1에서는 이를 다시 복구할 수 있는 방안을 제공한다.  

하지만 NameNode의 장애로 인한 단일 지점 실패(SPOF, Single Point Of Failure)는 막을 수 없었다. NameNode 장비에 문제가 생기면 문제를 해결할 때까지는 전체 하둡 클러스터를 사용할 수 없는 것이다. 하둡 2부터는 고가용성(HA, High Availability) 기능이 제공되는데, Active-Standby 두 개의 NameNode를 구성함으로서 NameNode 장애로 인한 SPOF를 막을 수 있게 되었다.


### Secondary NameNode
하둡 1에서 제공하는 NameNode 장애시 데이터 복구를 위한 노드이다. NameNode의 FsImage와 EditLog를 주기적으로 폴링하여 백업하고, 쌓여있는 EditLog를 FsImage에 병합(merge)한다. 병합된 변경 사항은 다시 NameNode로 전달하여 동기화한다. 이러한 과정을 체크포인트라고 한다. CPU와 IO가 많이 필요한 작업이기 떄문에 별도의 노드에서 수행하는 것이다.하둡 2에서도 HA 구성을 활성화 하지 않으면 Secondary NameNode를 FsImage와 EditLog의 백업 및 병합 용도로 사용할 수 있다.

![Secondary NameNode](/hadoop-namenode-3.png) _그림 3. 체크포인트. SNN는 EditLog와 FsImage를 백업하고 EditLog를 병합하는 역할을 한다._

설명과 그림은 간단하게 표현햤지만 Secondary NameNode는 NameNode에게 RPC와 여러 번 HTTP 요청을 해서 체크포인트를 진행한다.  
1. NameNode 이미지의 트랜잭션 아이디를 확인  
2. NameNode의 EditLog를 rolling 하도록 요청  
3. NameNode의 FsImage와 EditLog를 가져옴  
4. FsImage 확인 후 네임스페이를 새로 갱신  
5. EditLog 적용  
6. 새로운 FsImage 생성  
7. NameNode로 새로운 이미지 업로드  


### Standby NameNode
두 개의 NameNode를 구성하여 Active/Standby 상태를 갖게 하는데, 두 NameNode는 FsImage와 EditLog 저장을 위해 NFS와 같은 공유 저장소를 설정한다. Active 노드가 파일을 변경하면, Standby 노드가 감시(watch)하고 있다가 변경 사항을 자신의 네임스페이스에 똑같이 적용한다. 이렇게 동기화된 상태의 네임스페이스를 유지하여, Active 노드에 장애가 발생했을 때 Standby 노드가 Active 노드의 역할을 곧바로 수행한다. 이로서 SPOF 문제를 해결할 수 있다. 

체크포인트는 Secondary NameNode에서 수행하는 것 보다 간결해졌다. Standby 노드는 공유 저장소의 EditLog를 계속해서 본인의 네임스페이스에 적용하므로, 네임스페이스를 새로운 FsImage로 저장하고 Active 노드에게 새로운 FsImage를 사용하도록 함으로서 완료된다.

![Standby NameNode with NFS](/hadoop-namenode-4.png) _그림 4. Active/Standby NameNode with NFS_



### JournalNode
NFS와 같은 공유 저장소를 사용하는 방법은, 다시 이 공유 저장소의 장애로부터 자유롭지 못한 문제점을 낳는다. 공유 저장소 자체가 HA 구성이 되어있지 않으면 이 저장소가 SPOF가 될 수 있고 이로인해 전체 클러스터의 장애로 이어진다. 

이 문제를 해결하기 위해 단일 공유 저장소 대신 NameNode에 JournalManager를 두고 JournalNode로 이루어진 분산 저장소에 FsImage와 EditLog를 저장하고 동기화하여 완벽히 HA를 보장할 수 있게 되었다.

![Standby NameNode with JournalNode](/hadoop-namenode-5.png) _그림 5. Active/Standby NameNode with JournalNode_

---