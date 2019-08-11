---
author: "Kim, Geunho"
date: 2016-03-29
title: "Nodejs - Error: listen EACCES 0.0.0.0:80"
---


# 발생 원인 및 해결 방법
root 권한이 없는 계정으로 Linux 내에서 애플리케이션을 수행할 때, 1024 보다 작은 포트에는 바인딩 할 수 없다. Nodejs에서 웹 서버를 띄울 때에도 마찬가지 인데, 아래와 같은 방법으로 처리 할 수 있다.

1. 애플리케이션을 root 권한으로 수행
2. 혹은, 1024 이상의 포트 번호를 바인딩 하도록 수정
3. 애플리케이션이 1024 보다 작은 포트에 바인딩 할 수 있도록 설정

이 중 세 번째 방법은 애플리케이션에 cap_net_bind_service capability를 설정하면 된다.

```
# root 계정으로 로그인
su –

# NODE_PATH 변수에 nodejs 파일 경로 저장
NODE_PATH=$(which nodejs)

# nodejs의 capability 설정
setcap 'cap_net_bind_service=+ep' $NODE_PATH

# capability 설정 확인
getcap $NODE_PATH
/usr/bin/nodejs = cap_net_bind_service+ep
```

Nodejs 뿐만 아니라 다른 애플리케이션도 위와 같은 방법으로 1024 미만 포트에 root 권한 없이 바인딩이 가능하다. 

* 주의: 파일의 접근 권한을 변경하게 되면 설정한 capability가 제거된다.