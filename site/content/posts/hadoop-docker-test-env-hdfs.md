---
author: "Kim, Geunho"
date: 2019-08-28
title: "Docker로 Hadoop 테스트 환경 구축하기 - HDFS"
---

# Docker와 테스트 환경
Docker는 애플리케이션을 컨테이너화해서 이미지로 만들어 배치하고 실행할 수 있는 환경을 제공하는 컨테이너 도구[^1]이다.  
격리된 파일 시스템을 갖는 컨테이너 특성 덕분에, 운영 환경에 대한 프로비져닝 뿐만 아니라 Docker가 설치된 환경에서 컨테이너화된 애플리케이션을 즉시 구동시켜서 테스트할 수 있는 환경을 구성하는데에도 유용하게 쓰인다.  

오픈소스를 포함한 다양한 애플리케이션을 로컬 환경에 설치하고 구성하면서 테스트하다보면(주로 개발자의 노트북), 어느새 로컬 환경은 지저분해지고 몇 해 전에 설치한 애플리케이션이 자동 재시작되어 리소스를 점유하게 된다. 환경 설정 파일은 언제 어떻게 바꿨는지도 모른다. VM 환경으로 OS를 완전히 격리시켜 구성하자니 구동하는데 시간이 오래 걸리고 구동한 이후 점유하는 리소스가 노트북이 감당하기에는 참 크다.  

Docker는 이런 환경에서 최적의 도구이다. 컨테이너를 어떻게 구성할지 애플리케이션마다 설정 파일을 생성하고, 서로 상호작용하는 애플리케이션인 경우 다시 실행과 관련된 설정 파일을 생성해서 여러 개의 애플리케이션을 순식간에 실행하고 다시 종료시킬 수 있다.  

이 포스트에서는 Hadoop HDFS를 Docker로 컨테이너화 헤서 로컬에 테스트 환경으로 구성하는 방법을 설명한다.  

<br/>

# HDFS
하둡 분산 파일 시스템(HDFS)은 마스터 노드인 [NameNode](/posts/hadoop-namenode)와 워커 노드인 [DataNode](/posts/hadoop-datanode)로 이루어져있다. 각각 JVM 위에서 동작하는 자바 애플리케이션으로, 두 애플리케이션만 동작을 하면 HDFS를 구성해서 읽고 쓸 수 있다.  
실행 환경은 일반적은 GNU/Linux 그리고 Windows 환경을 지원하며, Hadoop 2.7~2.x[^2] 버전은 Java 7, 8을 지원한다.[^3]  

클라이언트에서 각 애플리케이션과 직접 통신하는 포트 정보는 다음 표와 같다.


| 애플리케이션 &nbsp;&nbsp; | 기본포트 &nbsp;&nbsp; | 프로토콜 &nbsp;&nbsp; | 상세설명                               | 관련 설정 항목                  |
|------------------------|---------------------|--------------------|--------------------------------------|-----------------------------|
| NameNode               | 50070               | HTTP                | HDFS 상태 및 파일 시스템 조회용 Web UI &nbsp;   | `dfs.http.address`    |
|                        | 9000                | IPC                 | 파일 시스템 메타데이터 관련 작업용          | `fs.defaultFS`              |
| DataNode               | 50075               | HTTP                | 상태 및 로그 조회용 Web UI              | `dfs.datanode.http.address` |
|                        | 50010               | TCP                 | 데이터 전송                            | `dfs.datanode.address`      |
_표 1. 클라이언트에서 HDFS와 직접 통신할 때 사용하는 포트 정보_

이 자료들을 가지고 이제 본격적으로 Docker 컨테이너화를 진행해보자.

<br/>

# Dockerize
애플리케이션을 Docker에서 사용하는 설정 파일인 `Dockerfile` 양식에 맞게 구성한다. 이 설정 파일이 있으면 Docker 이미지 빌드가 가능하고, 생성한 이미지를 어디에서든 당겨와서(pull) 실행할 수 있다.  

사실 이미지를 생성하는 것은 이 설정 파일 없이 직접 기반이 되는 OS 이미지로부터 컨테이너를 실행해서 쉘로 접속, 필요한 작업을 하고 이미지를 커밋해도 된다.  
하지만 이 방식은 다음과 같은 문제점이 있다:  

* 이미지가 어떻게 생성되었는지 기록이 남지 않음  
* 따라서 작업한 내용을 다시 되돌리거나, 몇 가지 설정을 변경해서 새로운 이미지를 생성하기 어려움  
* 그리고 생성된 이미지를 신뢰하기 어려움. 따라서 다른 사람과 공유하는데에도 적합하지 않음  

블록을 쌓아 올리듯 이미지를 만드는 Docker 세계에서는, 처음에는 조금 번거로워 보이더라도 `Dockerfile`로 이미지를 만드는 것에 익숙해지는 것이 좋다.  

## Base
NameNode 이미지를 만들기 전에, 먼저 각 컴포넌트의 밑바탕이 되는 이미지를 구성한다. 밑바탕 이미지는 다음과 같은 내용을 포함헤야 할 것이다.

* Hadoop binary  
* Java  
* Common configurations  

여기에서는 현재 2019년 8월 기준 Hadoop 2버전의 최신 버전인 2.9.2 버전을 사용했다. Hadoop 2는 Java 7 이상을 지원하므로 Java 8을 선택했다. 마지막으로 공용 설정은 core-site.xml, hdfs-site.xml을 포함하도록 구성할 것이다.  

먼저 다음과 같은 디렉토리 구조로 폴더와 파일을 생성한다.  
```
.
└── hadoop
    ├── base
    │   ├── Dockerfile
    │   └── core-site.xml
    ├── namenode
    │   ├── Dockerfile
    │   ├── hdfs-site.xml
    │   └── start.sh
    └── datanode
        ├── Dockerfile
        ├── hdfs-site.xml
        └── start.sh
```

hadoop의 하위 폴더인 base, namenode, 그리고 datanode 폴더와 그 하위에 다시 위치한 개별 Dockerfile은 각 항목이 이미지로 빌드될 것임을 보여준다.  

`baes/Dockerfile`의 내용을 다음과 같이 작성한다. 

```dockerfile
# 기반이 되는 이미지를 설정. 여기서는 ubuntu 18 기반인 zulu의 openjdk 8버전을 선택
FROM azul/zulu-openjdk:8

# 바이너리를 내려받기 위해 설치
RUN apt-get update && apt-get install -y curl 

ENV HADOOP_VERSION=2.9.2
ENV HADOOP_URL=http://mirror.apache-kr.org/hadoop/common/hadoop-$HADOOP_VERSION/hadoop-$HADOOP_VERSION.tar.gz

# Hadoop 2.9.2 버전을 내려받고 /opt/hadoop에 압축 해제
RUN curl -fSL "$HADOOP_URL" -o /tmp/hadoop.tar.gz \
    && tar -xvf /tmp/hadoop.tar.gz -C /opt/ \
    && rm /tmp/hadoop.tar.gz

# 데이터 디렉토리 생성 및 설정 폴더의 심볼릭 링크 생성
RUN ln -s /opt/hadoop-$HADOOP_VERSION /opt/hadoop \
    && mkdir /opt/hadoop/dfs \
    && ln -s /opt/hadoop-$HADOOP_VERSION/etc/hadoop /etc/hadoop \
    && rm -rf /opt/hadoop/share/doc

# 로컬의 core-site.xml 파일을 복제
ADD core-site.xml /etc/hadoop/

# 실행 환경에 필요한 환경 변수 등록
ENV HADOOP_PREFIX /opt/hadoop
ENV HADOOP_CONF_DIR /etc/hadoop
ENV PATH $HADOOP_PREFIX/bin/:$PATH
ENV JAVA_HOME /usr/lib/jvm/zulu-8-amd64
```

기반 이미지인 `azul/zulu-openjdk:8`는 ubuntu 18 기반이라서 이미지 크기가 315MB로 꽤 크다.  
이 크기를 작게 줄이려고 alpine 기반으로 생성해서는 안된다. Hadoop은 일반적인 GNU/Linux 환경에서 동작할 수 있도록 개발되었기 때문이다.  

`base/core-site.xml`은 다음과 같이 작성한다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://namenode:9000/</value>
    <description>NameNode URI
    </description>
  </property>  
</configuration>
```
`fs.defaultFS` 설정은 NameNode 위치를 찾는 중요한 설정으로, DataNode 뿐만 아니라 각종 애플리케이션에서 NameNode에 파일 읽기/쓰기 요청을 할 때 사용되는 항목이다.  
URI의 호스트 이름이 `namenode`로 설정되었는데, NameNode 컨테이너의 호스트 이름을 `namenode`로 지정할 것이다.  

이제 터미널에서 `base` 디렉토리로 이동해서 baes 이미지를 생성해보자.

```sh
cd base
docker build -t hadoop-base:2.9.2 .
```

현재 위치한 폴더의 Dockerfile을 사용해서 docker 이미지를 빌드하겠다는 명령어이다. Dockerfile을 작성한대로 각 라인이 하나의 빌드 단계가 되어 명렁어를 실행하는 로그가 출력된다.  

빌드가 완료되면 다음 명렁어로 로컬에 생성된 이미지를 확인할 수 있다.  
```sh
docker images

REPOSITORY        TAG                 IMAGE ID            CREATED             SIZE
hadoop-base       2.9.2               fab79eab7d71        2 mins ago          1.2GB
```
이 이미지는 다음과 같은 이미지 계층 구조를 갖는다. 간단한 구조를 갖지만 이 base 이미지를 통해 모든 Hadoop 컴포넌트의 기반으로 사용할 수 있다.  

![Hadoop base 이미지 계층](/hadoop-docker-test-env-hdfs-1.png) _그림 1. Hadoop base 이미지 계층 구조. OS → Runtime → App base의 간단한 구조이다._

<br/>

## NameNode
Java와 Hadoop binary를 가지고 있는 base 이미지를 생성했으니, 이제 NameNode 이미지를 생성해보자.  
NameNode 구성시 고려해야할 부분은 다음과 같다.  

* FsImage, EditLog를 저장하는 로컬 파일 시스템 경로  
* NameNode용 `hdfs-site.xml` 설정  
* NameNode 최초 구동 확인 및 네임스페이스 포맷

먼저 `hdfs-site.xml` 파일을 다음과 같이 작성한다.  
`dfs.namenode.name.dir`은 FsImage, EditLog 파일을 저장하는 경로이다. HDFS 파일 블록 크기를 결정하는 `dfs.blocksize` 항목은 바이트 수로 여기서는 테스트를 위해 기본 값보다 훨씬 작은 10MB로 설정했다. (기본값은 128MB) 이 외의 항목들은 [hdfs-default.xml](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml) 파일을 참고한다.  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///opt/hadoop/dfs/name</value>
  </property>
  <property>
    <name>dfs.blocksize</name>
    <value>10485760</value>
  </property>
  <property>
    <name>dfs.client.use.datanode.hostname</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-bind-host</name>
    <value>0.0.0.0</value>
  </property>
  <property>
    <name>dfs.namenode.servicerpc-bind-host</name>
    <value>0.0.0.0</value>
  </property>
  <property>
    <name>dfs.namenode.http-bind-host</name>
    <value>0.0.0.0</value>
  </property>
  <property>
    <name>dfs.namenode.https-bind-host</name>
    <value>0.0.0.0</value>
  </property>
</configuration>
```

시작 스크립트인 `start.sh`에서는 NameNode의 네임스페이스가 포맷되었는지 확인하고, 포맷되기 전이라면 구동 전 포맷을 먼저 진행해야 한다.  
다음과 같이 작성했다.  
```sh
#!/bin/bash

# 네임스페이스 디렉토리를 입력받아서 
NAME_DIR=$1
echo $NAME_DIR

# 비어있지 않다면 이미 포맷된 것이므로 건너뛰고
if [ "$(ls -A $NAME_DIR)" ]; then
  echo "NameNode is already formatted."
# 비어있다면 포맷을 진행
else
  echo "Format NameNode."
  $HADOOP_PREFIX/bin/hdfs --config $HADOOP_CONF_DIR namenode -format
fi

# NameNode 기동
$HADOOP_PREFIX/bin/hdfs --config $HADOOP_CONF_DIR namenode
```

Dockerfile은 base 이미지 덕분에 간단하다.
```dockerfile
# 방금 전 로컬에 생성한 base 이미지
FROM hadoop-base:2.9.2

# NameNode Web UI 응답 여부를 통해 Healthcheck
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 CMD curl -f http://localhost:50070/ || exit 1

# 설정 파일 복제
ADD hdfs-site.xml /etc/hadoop/

# FsImage, EditLog 파일 경로를 volume으로 연결
RUN mkdir /opt/hadoop/dfs/name
VOLUME /opt/hadoop/dfs/name

# 실행 스크립트 복제
ADD start.sh /start.sh
RUN chmod a+x /start.sh

# NameNode의 HTTP, IPC 포트 노출
EXPOSE 50070 9000

# 시작 명령어 등록
CMD ["/start.sh", "/opt/hadoop/dfs/name"]
```

이제 이미지를 생성하자. 기본 명령어와 바이너리 파일을 다운로드 받는 등 시간이 오래 걸리는 작업은 이미 base 이미지에서 처리했기 때문에, 그 위에 다시 한번 계층을 쌓는 NameNode는 빌드 시간이 얼마 걸리지 않는 것을 볼 수 있다.  
```sh
cd namenode
docker build -t hadoop-namenode:2.9.2 .
```

두 개의 이미지가 생성되었고 계층 구조에 하나의 이미지가 더해졌다.  

![Hadoop namenode 이미지 계층](/hadoop-docker-test-env-hdfs-2.png)

NameNode 이미지를 생성했으니 이제 실제로 기동시켜볼 차례이다. 어차피 DataNode 이미지를 생성하면 docker compose를 구성하는 것이 훨씬 간편하므로 바로 compose 설정 파일을 구성하겠다.  

먼저 root 디렉토리에 docker-compose.yml 파일을 생성한다. 디렉토리 구조를 보면 아래와 같다.
```
.
└── hadoop
    ├── base
    ├── namenode
    ├── datanode
    └── docker-compose.yml
```

설정 파일의 내용을 작성한다. `services`에는 기동할 컨테이너 정보를, `volumes`에는 마운트할 volume 정보를, `networks`에는 컨테이너 간 어떤 네트워크 구성을 사용할지 지정하는 항목이다.  
`volumes` 이하에 정의된 각 항목은 다음과 같은 의미를 갖고 있다.  

* `namenode:/opt/hadoop/dfs/name`: `namenode` volume을 컨테이너의 `/opt/hadoop/dfs/name` 경로에 마운트  
* `/tmp:/tmp`: 컨테이너가 기동되는 호스트의 `/tmp` 경로를 컨테이너의 `/tmp` 경로로 마운트. 호스트와 컨테이너 간 파일을 쉽게 공유하기 위해서 지정함  

콜론으로 구분된 설정 값의 뒷 부분이 컨테이너에 해당한다는 것을 기억하면 쉽게 이해할 수 있다. 

```yml
version: "3"

services:
  namenode:
    image: hadoop-namenode:2.9.2
    container_name: namenode
    hostname: namenode
    ports:
      - "50070:50070"
      - "9000:9000"
    volumes:
      - namenode:/opt/hadoop/dfs/name
      - /tmp:/tmp
    networks:
      - bridge

volumes:
  namenode:

networks:
  bridge:
```

이제 이 설정 파일을 바탕으로 NameNode 컨테이너를 기동할 수 있다.  
```sh
# docker-compose.yml 파일이 위치한 경로로 이동
cd hadoop

# 컨테이너를 백그라운드에서 실행
#  종료할 때에는 docker-compose down
docker-compose up -d
Creating namenode        ... done

# 기동중인 컨테이너 프로세스 확인
docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS                            PORTS               622e4f642435
hadoop-namenode:2.9.2          "/start.sh /opt/hado…"   10 seconds ago      Up 6 seconds (health: starting)   0.0.0.0.:9000->9000/tcp, 0.0.0.0.:50070->50070/tcp 

# 생성된 bridge 네트워크 확인
docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
0d9989bd9f78        bridge              bridge              local
9ae1ae84360f        hadoop_bridge       bridge              local
62a94ebd58b7        host                host                local

# 생성된 volume 확인
docker volume ls
DRIVER              VOLUME NAME
local               hadoop_namenode

# namenode 컨테이너의 로그 확인
docker logs -f namenode
```

docker compose로 기동한 후 마지막 명령어로 로그를 살펴보면, 네임스페이스를 포맷한 로그와 NameNode를 기동한 로그를 확인할 수 있다.  
브라우저에서 http://localhost:50070 으로 접속하면 아래와 같은 NameNode Web UI에 접근한다.  

![NameNode Web UI](/hadoop-docker-test-env-hdfs-3.png)

또한 NameNode가 기동되었으니, 아직 DataNode를 띄우지 않더라도 폴더의 생성과 삭제, 그리고 파일 목록 조회가 가능하다.  
NameNode 컨테이너 내부에 있는 Hadoop 클라이언트를 실행해서 테스트해보자.  
```sh
# namenode 이름의 컨테이너의 hadoop 클라이언트를 실행. 파일 시스템의 root 디렉토리를 모두 조회한다.
docker exec namenode /opt/hadoop/bin/hadoop fs -ls -R /

# 명령어 등록
alias hadoop="docker exec namenode /opt/hadoop/bin/hadoop"

# 폴더 생성/조회/삭제
hadoop fs -mkdir -p /tmp/test/app
hadoop fs -ls -R /tmp
hadoop fs -rm -r /tmp/test/app
```

`docker exec [컨테이너 이름] [명령어]`를 입력하면 동작 중인 컨테이너 내에서 `[명령어]`에 해당하는 명령어를 실행한 결과의 표준 출력을 현재 쉘에 출력한다.  
컨테이너 내 자주 실행하는 프로그램응 `alias`로 등록해두면 편리하다. 여기에서는 NameNode의 `bin/hadoop` 프로그램을 `hadoop`으로 alias 지정했다.  

## DataNode
먼저 작성했던 JVM과 하둡 바이너리를 포함하는 base 이미지로부터 DataNode 이미지를 생성해보자. 고려해야할 부분은 두 가지이다.  
* 파일 블록을 저장하는 로컬 파일 시스템 경로  
* DataNode용 `hdfs-site.xml` 설정  
파일 블록 저장 경로는 `$HADOOP_PREFIX/dfs/data`로 설정할 것이다. 이 내용은 `hdfs-site.xml` 파일을 `datanode` 디렉토리 이하에 생성하고 내용을 다음과 같이 작성한다.  

```xml
<configuration>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///opt/hadoop/dfs/data</value>
  </property>
  <property>
    <name>dfs.blocksize</name>
    <value>10485760</value>
  </property>  
  <property>
    <name>dfs.datanode.use.datanode.hostname</name>
    <value>true</value>
  </property>
</configuration>
```

시작 스크립트는 `start.sh`에 다음과 같이 작성한다.  
```sh
#!/bin/sh

$HADOOP_PREFIX/bin/hdfs --config $HADOOP_CONF_DIR datanode
```
`hdfs` 프로그램을 `datanode` 옵션으로 시작하면 DataNode 프로세스가 기동한다.  

마지막으로 Dockerfile 이다. `HEALTHCHECK`에 Web UI 주소가 입력된 것과 `EXPOSE`에 명시한 포트를 눈여겨 본다.  

```dockerfile
FROM hadoop-base:2.9.2

# DataNode Web UI 응답 여부를 통해 Healthcheck
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3  CMD curl -f http://localhost:50075/ || exit 1

RUN mkdir /opt/hadoop/dfs/data
VOLUME /opt/hadoop/dfs/data

ADD start.sh /start.sh
RUN chmod a+x /start.sh

# WebUI, 데이터전송
EXPOSE 50075 50010

CMD ["/start.sh"]
```

NameNode 이미지를 구성할 때와 크게 다르지 않다. 이제 생성한 파일들을 가지고 이미지를 생성한다.  
```sh
cd datanode
docker build -t hadoop-datanode:2.9.2 .
```

이제 컨테이너 실행 환경 정보를 담고 있는 `docker-compose.yml` 파일에 DataNode를 추가한다. 로컬 환경이지만 세 개의 DataNode를 띄울 것이다.  
```yml
version: "3.4"

# 이미지와 네트워크 정보에 대한 base service를 지정
x-datanode_base: &datanode_base
  image: hadoop-datanode:2.9.2
  networks:
    - bridge

services:
  namenode:
    image: hadoop-namenode:2.9.2
    container_name: namenode
    hostname: namenode
    ports:
      - 50070:50070
      - 9000:9000
    volumes:
      - namenode:/opt/hadoop/dfs/name
      - /tmp:/tmp
    networks:
      - bridge

  datanode01:
    <<: *datanode_base
    container_name: datanode01
    hostname: datanode01
    volumes:
      - datanode01:/opt/hadoop/dfs/data
  
  datanode02:
    <<: *datanode_base
    container_name: datanode02
    hostname: datanode02
    volumes:
      - datanode02:/opt/hadoop/dfs/data

  datanode03:
    <<: *datanode_base
    container_name: datanode03
    hostname: datanode03
    volumes:
      - datanode03:/opt/hadoop/dfs/data

volumes:
  namenode:
  datanode01:
  datanode02:
  datanode03:

networks:
  bridge:
```

파일을 수정하고 `docker-compose up -d`를 다시 실행하면 세 개의 DataNode 컨테이너를 추가로 띄우게 된다. DataNode는 NameNode로 heartbeat을 보내면서 NameNode에 등록된다.

NameNode

## Proxy


---
이어지는 포스팅에서는 HDFS 파일을 읽고 쓰는 예제와 Proxy 구성을 통해 로컬 환경에서 reverse proxy로 서로 다른 DataNode 컨테이너의 Web UI에 접근하는 방법을 설명할 예정이다.  


[^1]: Docker 공식 홈페이지의 [What is a Container?](https://www.docker.com/resources/what-container)를 참고한다.  
[^2]: 이 글을 작성하는 시점의 Hadoop 2 최신 버전인 2.9.2 버전의 공식 문서를 참고했다.  
[^3]: 그 외 지원 버전은 Hadoop confluence 페이지의 [Hadoop Java Versions](https://cwiki.apache.org/confluence/display/HADOOP/Hadoop+Java+Versions)를 참고한다. 