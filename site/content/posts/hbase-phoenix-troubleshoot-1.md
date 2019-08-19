---
author: "Kim, Geunho"
date: 2016-06-17
title: "Hbase, Phoenix - PhoenixIOException: The system cannot find the path specified"
---


# 발생원인 및 해결방법
윈도우에서 Phoenix에 jdbc client로 접속 후 `order by` 혹은 `group by`가 포함된 쿼리를 수행하면 아래와 같은 에러 메시지가 발생한다.

```
PhoenixIOException: The system cannot find the path specified
```

메시지 내용 그대로 특정 파일 경로를 찾지 못해 발생하는 IOException인데, 특이한 점은 위와 같은 집계 절이 포함된 쿼리에 대해서만 발생 한다는 것이다. 이는 Phoenix의 쿼리 처리 방법과 관련이 있다. Phoenix 쿼리 서버에 쿼리를 요청하면,

1. Hbase 서버의 데이터를 scan (full or range)
2. 필요한 경우 client에서 데이터 처리 (merge, limit, …)  

를 하게 된다. 이는 explain 명령어를 통해 쿼리 실행 계획을 보고 알 수 있다.

```sql
explain
select host, count(host) as cnt from logs
where timestamp > to_date(‘2016-06-17’)
group by host
order by host

PLAN
——————————————
CLIENT 2-CHUNK PARALLEL 2-WAY RANGE SCAN  – (1)
SERVER AGGREGATE INTO DISTINCT ROWS  – (2)
CLIENT MERGE SORT  – (3)
```

클라이언트에서 범위 스캔 후 (1) 서버에서 집계를 하고 (2) 마지막으로 다시 클라이언트에서 병합 정렬을 (3) 수행한다.  

마지막 (3) 실행시 임시 폴더를 사용하게 되는데 이 값이 4.6.0 이하 버전까지는 `/tmp` 라는 정적 상수 값으로 설정되어 있었다. (`static final String DEFAULT_SPOOL_DIR = "/tmp"`)  
이는 https://issues.apache.org/jira/browse/PHOENIX-2348 에서 보고되었고 4.7.0 버전에서 수정되었는데, 이전 버전의 Phoenix client를 윈도우 환경에서 사용할 경우 위와 같이 에러가 발생할 수 있다.

결국 해결 방법은 간단한데, client가 실행되는 위치의 드라이브에 /tmp 폴더를 생성해주면 된다.

