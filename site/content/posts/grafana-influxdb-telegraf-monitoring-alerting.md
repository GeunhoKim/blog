---
author: "Kim, Geunho"
date: 2017-01-02
title: "Grafana, Influxdb, Telegraf - 서버의 관제(Monitoring)와 알림(Alerting)"
---


# 서버의 관제(Monitoring)에 대해서
개발자의 DevOps 역량이 점점 중요해지면서, 서버의 운영에도 적극적으로 개입하게 되었다. 단순히 인프라 부서에서 일괄적인 지표로 관제하고 문제가 생겼을 때 수동적으로 통보 받는 것에서 벗어나, 서버의 리소스 뿐만 아니라 서비스에서 발생하는 다양한 지표들을 관제하여 서비스의 장애 여부를 즉시 알아차릴 수 있어야 한다.  

이 글에서는 1) 오픈소스로 공개된 influxdata의 tsdb(Time Series DataBase)인 [influxdb](https://github.com/influxdata/influxdb)와 collector agent인 [telegraf](https://github.com/influxdata/telegraf)를 통해 시계열 데이터를 적재하고, 2) 오픈소스 대시보드인 [grafana](https://github.com/grafana/grafana)를 통해 관제 대시보드와 알림 통보(alert)를 구성하여, 3) 특정 상황에서 사용자가 정의한 동작을 수행하도록 하는 방법을 설명한다.

# 시스템 구조
먼저, 각 서버에는 서비스와 telegraf 서비스가 동작한다. telegraf는 서비스와 운영체제의 리소스를 모아서(collect) influxdb에 적재한다. influxdb는 서비스 그룹당 하나의 인스턴스로 구성한다. 마지막으로 grafana를 통해  influxdb의 데이터를 조회하여 대시보드로 구성한다. 관제 대상 중 grafana의 alert 기능을 통해 slack으로 통보하거나, webhook을 통해 개발자가 정의한 동작을 수행할 수 있다. 

![System architecture](/grafana-influxdb-telegraf-monitoring-alerting-1.png) _그림 1. 시스템 구성도_

데이터 흐름과 동작은 다음과 같다:

1. telegraf에서 influxdb로 운영 체제 리소스 데이터 적재
2. telegraf에서 influxdb로 서비스 지표 적재
3. grafana에서 influxdb를 읽어들여 차트와 대시보드 구성
4. grafana에서 구성한 차트에 대해 slack 통보 추가
5. grafana에서 구성한 차트에 대해 webhook 추가

이번 포스팅에서는 2번 항목의 telegraf 대신 서비스에서 지표를 직접 적재하는 방식으로 구성[^1]할 것이다.

# 시작하기
## 서비스 endpoints
로컬 mac os에서 시스템을 구성하는 방법을 설명한다. 먼저 아래와 같은 구성으로 endpoint를 구성할 것이다.

* Service A: http://localhost:8080
* influxdb: http://localhost:8086
* grafana: http://localhost:80
여기서 Service A 는 nodejs 런타임에서 동작하는 rest api 서버이다. `GET /api/hello` 하나의 api를 서비스하며, api 호출 수와 지연 시간을 influxdb로 전송한다. Service A의 전체 코드는 [여기](https://github.com/geunho/express-influx)에서 확인할 수 있다.  

## influxdb
앞서 간단히 소개한 것 처럼, influxdb는 시계열 데이터베이스(Time Series DataBase, TSDB)이다. golang으로 작성된 이 데이터베이스의 데이터는 유닉스 시간(흔히 timestamp라고 부르는)과 그 시간에 해당하는 복수 개의 숫자로 된 필드 값, 그리고 태그로 이루어져 있다.

몇 가지 주요 특징들은 다음과 같다:

* 데이터베이스는 golang으로 작성되어 단일 바이너리로 제공됨. 외부 의존성이 없음. 즉, 설치와 환경설정이 간편함
* HTTP API 를 통해 읽기/쓰기 동작이 이루어짐. 간편한 인터페이스의 읽기/쓰기
* SQL 구문과 같은 조회 및 집계 가능 (e.g. 태그를 이용한 where clause)
* 자동 리텐션 정책을 지원
* HA, Relay (buffering, sharding, recovering) 기능 지원
* HA, Clustering 지원 (단, 유료 버전에서만 지원됨[^2])

과거 cacti와 같은 서비스에서 많이 사용됐던 RRDTool(Round Robin Database Tool)과 달리 다른 서비스에서 http 프로토콜을 통해 손쉽게 데이터를 적재할 수 있고, 집계 데이터를 읽어서 다양한 서비스로 연동할 수 있다. (e.g. anomaly detection)  
더 자세한 내용은 [공식 문서](https://docs.influxdata.com/influxdb/v1.1)를 참고한다.

```bash
# homebrew를 통해 설치
$ brew update
$ brew install influxdb
==> Downloading https://homebrew.bintray.com/bottles/influxdb-1.1.1.sierra.bottle.tar.gz
…
To have launchd start influxdb now and restart at login:
  brew services start influxdb
Or, if you don’t want/need a background service you can just run:
  influxd -config /usr/local/etc/influxdb.conf
==> Summary
  /usr/local/Cellar/influxdb/1.1.1: 24B
```

이 글을 작성하는 시점에는 1.1.1 버전이 homebrew에 릴리즈 되어 있었다. 설치 로그를 살펴보면, brew 명령어를 통해 간단히 서비스를 시작할 수 있으며 `/usr/local/etc/influxdb.conf`가 구성 파일의 경로이며, `/usr/local/Cellar/influxdb/1.1.1`는 바이너리 경로임을 알려주고 있다. 구성 파일은 열어서 한번 훑어보는 것을 추천한다. (우리는 로컬에 테스트할 목적이므로 특별히 수정할 부분은 없다. http endpoint 포트가 기본 값으로 8086임을 확인할 수 있다)

```bash
# homebrew를 통해 서비스 구동
$ brew services start influxdb
…
==> Successfully started `influxdb` (label: homebrew.mxcl.influxdb)

# 서비스 동작 확인
$ brew services list
Name     Status  User Plist
influxdb started {username} /Users/{username}/Library/LaunchAgents/homebrew.mxcl.influxdb.plist

# influx shell 접속
$ influx
…
Connected to http://localhost:8086 version v1.1.1
InfluxDB shell version: v1.1.1
> show databases
name: databases
name
—-
_internal
> use _internal
> show measurements
name: measurements
name
—-
cq
database
httpd
queryExecutor
runtime
…
```

처음 influxdb shell에 접속하면 _internal 데이터베이스 한 개만 존재하는데, 이름에서 알 수 있듯이 influxdb 내부용 데이터베이스이다. 그 안의 measurements에는 메타데이터 뿐만 아니라 influxdb 서비스 동작 중 리소스 사용 현황까지 적재된다. 맛보기로 하나의 행을 조회해보자.

```bash
> select * from runtime limit 1
name: runtime
time Alloc Frees HeapAlloc HeapIdle HeapInUse HeapObjects HeapReleased HeapSys Lookups Mallocs NumGC NumGoroutine PauseTotalNs Sys TotalAlloc hostname
—- —– —– ——— ——– ——— ———– ———— ——- ——- ——- —– ———— ———— — ———- ——–
1483495780000000000 3099184 43365 3099184 974848 4628480 26648 0 5603328 15 70013 1 19 182224 10230008 4819872 localhost
```

## telegraf
다음은 telegraf collector 구성 방법이다. 마찬가지로 golang으로 작성된 agent로, 설치와 구성 파일 배포가 간편하다.  

```bash
$ brew install telegraf
==> Downloading https://homebrew.bintray.com/bottles/telegraf-1.1.2.sierra.bottle.tar
…
To have launchd start telegraf now and restart at login:
  brew services start telegraf
Or, if you don’t want/need a background service you can just run:
  telegraf -config /usr/local/etc/telegraf.conf
…
```

서비스를 시작하기에 앞서, `/usr/local/etc/telegraf.conf` 파일을 열어서 살펴보자. 기본 값으로 설정되어 있는 `[agent]`, `[[outputs.influxdb]]` 항목은 로컬 환경 구성이므로 그대로 두어도 상관없다. 중요한 부분은 `[[inputs.*]]`으로 시작하는 항목들인데, [입력 플러그인 목록](https://github.com/influxdata/telegraf#input-plugins)을 참고하여 다양한 서비스의 시계열 데이터를 적재할 수 있다.  
`[[outputs.*]]`에서는 influxdb 뿐만 아니라 다양한 채널로 입력 값을 보낼 수 있다. (e.g. kafka와 같은 메시지 큐로 출력하여 실시간 처리에 이용)

일단 아무런 수정없이 서비스를 기동한다.
```bash
$ brew services start telegraf
```

이제 로컬 서버의 cpu, memory, process, … 와 같은 운영체제 관제에 필요한 기본적인 리소스 데이터가 influxdb로 유입되기 시작한다.  
influxdb shell에서 직접 조회해보자.  
```bash
$ influx
> show databases
name: databases
name
—-
_internal
telegraf
> use telegraf
```

_internal 데이터베이스 외에 telegraf 데이터베이스가 추가된 것을 볼 수 있다. (`telegraf.conf`의 `[[outputs.influxdb]]` 내에 있는 database 값을 수정하면 이름을 변경할 수 있음)  
measurements 목록을 살펴보자.
```bash
> show measurements
name: measurements
name
—-
cpu
disk
mem
processes
swap
system
```

현재부터 10분 전까지의 시스템의 cpu 사용량에 대해 30초 평균을 가져오는 쿼리이다.

```bash
> select mean(usage_system) from cpu where time > now() - 10m group by time(30s)
name: cpu
time mean
—- —-
1483535550000000000 5.112823832738451
1483535580000000000 4.554704323867991
1483535610000000000 4.156467778807397
1483535640000000000 5.1474029903729095
1483535670000000000 5.787787087879861
```

잘 쌓이고 있음을 확인 했으니, 이제 service 데이터베이스를 생성한 후 Service A를 기동하여 같은 방식으로 확인해본다.

```bash
> create database service
```

## Service A
Service A는 앞서 설명했던 것 처럼 nodejs 런타임에서 동작하는 아주 간단한 rest api 서버이다.

```
요청: GET /api/hello, 응답: {“hello”:”world!”}
요청: GET /api/world, 응답:{“world”:”hello!”}
```

nodejs를 설치하고 다음과 같이 내려받아 실행한다.
```bash
# 설치
$ git clone https://github.com/geunho/express-influx.git
$ cd express-influx/service-a
$ npm install
$ node app.js

# 실행
$ curl localhost:8080/api/hello
{“hello”:”world!”}
$ curl localhost:8080/api/world
{“world”:”hello!”}

# 몇 차례 더 요청을 날려서 데이터를 쌓아준다.
```

Service A는 요청시 요청수와 지연 시간을 influxdb로 기록한다. 데이터가 잘 쌓여있는지 확인한다.

```bash
$ influx
> use service
> select * from serviceA limit 5
name: serviceA
time host latency num_requests url
—- — —— ———— —
1483540807908284168 {hostname} 0.349809 1 /api/world
1483540807107210987 {hostname} 0.18984499999999999 1 /api/world
1483540765844455491 {hostname} 0.199207 1 /api/hello
1483540765697138565 {hostname} 0.22938699999999998 1 /api/hello
1483540765531034494 {hostname} 0.223208 1 /api/hello
```

서비스의 구성이 거의 완료 되었다. 이제 관제 대시보드 구성을 쉽게 할 수 있는 grafana를 설치하고 구성해보자.

## Grafana
Grafana는 elastic 사의 ELK(Elasticsearch, Logstash, Kibana) 제품 중 elasticsearch에 인덱싱된 데이터의 시각화와 대시보드 구성을 위한 도구인 Kibana에 영향을 받아 시작된 오픈 소스 프로젝트이다. Kibana를 써봤다면 어렵지 않게 grafana도 사용할 수 있다. 질의어를 통한 검색 기능을 제외하면, 시각화와 대시보드 기능은 Kibana에 비해 훨씬 강력하다.  

elasticsearch[^3], graphite, openTSDB, influxdb 등 다양한 시계열 데이터베이스를 데이터 소스로 사용하여, 하나의 그래프 박스 안에 여러 데이터베이스를 질의하여 구성할 수 있다. 특히 최근 정식 릴리즈된 4.0 버전부터 알림(alerting) 기능이 추가되어 데이터의 시각화 뿐만 아니라 관제용 대시보드로서 모든 기능을 갖추게 되었다.  

grafana를 설치해보자.
```bash
$ brew update
$ brew install grafana
==> Downloading https://homebrew.bintray.com/bottles/grafana-4.0.1.sierra.bottle.tar.
…
To have launchd start grafana now and restart at login:
  brew services start grafana
…
Or, if you don’t want/need a background service you can just run:
  grafana-server –config=/usr/local/etc/grafana/grafana.ini –homepath /usr/local/share/grafana cfg:default.paths.logs=/usr/local/var/log/grafana cfg:default.paths.data=/usr/local/var/lib/grafana cfg:default.paths.plugins=/usr/local/var/lib/grafana/plugins
```

다른 서비스들과 마찬가지로 homebrew를 이용하면 최신 버전의 정식 릴리즈를 쉽게 설치할 수 있다. 로그 출력을 통해 설정 파일의 경로가 `/usr/local/etc/grafana/grafana.ini` 임을 알 수 있다. grafana의 경우 앞서 80 포트로 endpoint를 구성하려 했으므로 해당 파일을 열어서 수정한다.

```bash
# /usr/local/etc/grafana/grafana.ini 수정
...
[server]
...
http_port = 80
root_url = http://localhost:80
...

# pie chart 플러그인 설치
$ grafana-cli plugins install grafana-piechart-panel
```

각 항목 앞의 세미콜론(;)은 주석 처리된 것을 뜻하므로 이를 제거해야 설정이 적용됨을 주의한다. 이제 서비스를 기동한다.  

```bash
$ sudo brew services start grafana
# default log 파일 경로 등의 권한 이슈가 있는 경우 sudo 권한으로 실행
```

브라우저에서 http://localhost를 주소창에 입력하면 grafana에 접속할 수 있다.

1. default 계정 및 패스워드로 `admin`/`admin` 입력 후 로그인
2. 좌측 상단의 아이콘을 클릭 > Data Sources 클릭 > 우측 상단의 Add data source 를 클릭
3. Name에 data source를 구분할 이름을 입력. 여기서는 resource로 정함. 우측에 default 체크박스 클릭
4. Type에 InfluxDB 선택
5. Url에 http://localhost:8086 입력
6. Database에 telegraf 입력 (telegraf.conf 에 설정된 telegraf 가 데이터 적재시 사용하는 database 임에 주의)
7. 그 외 설정은 그대로 둔 후 Add 버튼 클릭
8. 같은 방식으로 Database에 service  입력 후 Add 버튼 클릭 (나머지 설정은 동일)

여기까지 했다면 앞서 influxdb에 적재한 운영체제 리소스와 서비스 요청 지연시간에 대한 데이터 소스 생성이 끝난 것이다. http://localhost/datasources 접속시 아래와 같이 두 개의 data source가 보이면 된다.

![grafana datasources](/grafana-influxdb-telegraf-monitoring-alerting-2.png)

이제 대시보드를 구성할 차례이다. 여기서부터는 하나하나 설명하기 어려운 부분이 있으므로, 미리 구성한 대시보드를 import해서 직접 맛보길 바란다. 

* 좌측 상단 아이콘 클릭 > Dashboards 클릭 > import 클릭
* 혹은 paste JSON 텍스트 박스에 [예제](https://gist.github.com/geunho/237e6b2b1ca0ddab6a443fa2a4a71353) 내용을 그대로 붙여 넣고 입력 (가장 하단에 view raw 버튼을 클릭해서 새 탭에서 띄우면 전체 선택하기에 수월하다)
* telegraf/service 는 위에서 생성한 telegraf/service 입력


[^1]: 각 호스트에 agent인 telegraf를 설치해서 운영할 계획이라면, 서비스에서 직접 적재하는 것보다는 telegraf로 전달한 후, telegraf의 [influxdb_listener](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/influxdb_listener) 플러그인을 활용해 influxdb로 적재하도록 구성하는 것이 안전하다. 
[^2]: 이 글의 [시스템 구조](http://localhost:1313/posts/grafana-influxdb-telegraf-monitoring-alerting/#%EC%8B%9C%EC%8A%A4%ED%85%9C-%EA%B5%AC%EC%A1%B0)에서 서비스 그룹마다 influxdb를 구성하는 것을 제안하는 이유이기도 하다.
[^3]: 글을 작성하는 시점의 grafana 4.0 버전에서는 elasticsearch 5 버전을 지원하지 않았다. 릴리즈 전인 latest 베타 버전에서는 해당 [PR](https://github.com/grafana/grafana/pull/4970)이 머지되었는데, [4.1 부터 정식으로 지원](https://grafana.com/docs/guides/whats-new-in-v4-1/)한다.