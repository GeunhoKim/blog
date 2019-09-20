---
author: "Kim, Geunho"
date: 2019-09-20
title: "Docker for Mac 오류 - UNIX error exception: 17"
---

# 증상
여느 날처럼 맥에서 도커 데몬을 띄우고 작업을 시작하려는데 데몬이 크래시 리포트도 보여주지 않은채 죽어버렸다.  
몇 번 재시작을 해서 데몬이 기동되는 것을 확인했지만 역시 곧 종료되었다. 데몬에 무슨 일이 일어나는 것인지 알아보기 위해 실행 로그를 살펴보았다.  

# 로그 분석
맥에는 [Console](https://support.apple.com/ko-kr/guide/console/welcome/mac)이라는 실행 중인 프로세스에서 발생한 로그를 통합해서 살펴볼 수 있는 도구가 있다.  
Console 애플리케이션을 실행해보자.  
![Console](/docker-mac-crash-1.png)

상단의 검색 텍스트 박스에 `docker`를 입력하고 엔터를 치면 검색어가 포함된 프로세스 이름에서 발생한 로그들을 필터링하여 보여준다.  
이렇게 Console을 실행하고 다시 도커 데몬을 띄웠다.  

나의 경우 초기화 로그 이후 곧이어 아래와 같이 `docker-credential-osxkeychain` 프로세스에서 `UNIX error exception: 17` 오류가 연달아 발생한 후 데몬이 종료되었다.  
범인 발견! 프로세스 이름에서 유추해볼 때, 맥의 Keychain과 관련된 문제로 보인다.  
```
Type    Time                    Process                         Message    
default	10:00:50.086467 +0900	docker-credential-osxkeychain	UNIX error exception: 17
default	10:00:50.091450 +0900	docker-credential-osxkeychain	UNIX error exception: 17
default	10:00:50.094378 +0900	docker-credential-osxkeychain	UNIX error exception: 17
default	10:00:50.098576 +0900	docker-credential-osxkeychain	UNIX error exception: 17
default	10:00:50.101157 +0900	docker-credential-osxkeychain	UNIX error exception: 17
default	10:00:50.103548 +0900	docker-credential-osxkeychain	UNIX error exception: 17
default	10:00:50.206773 +0900	docker-credential-osxkeychain	UNIX error exception: 17
default	10:00:50.210874 +0900	docker-credential-osxkeychain	UNIX error exception: 17
```

# 원인 및 해결방법
이제 Keychain을 확인해보자. 만약 도커 설정 중 로그인 정보를 keychain에 등록하는 옵션이 활성화 되어있다면 아래와 같은 항목이 보일 것이다.
![Keychain](/docker-mac-crash-2.png)

상단 검색 박스에 `docker`를 입력하면 관련 항목만 리스팅된다. 

그런데 keychain에 등록되어 있어야할 로그인 정보가 보이지 않았는데, 도커에는 등록되어 있는것으로 판단하면서 충돌이 발생한 것으로 판단했다.  
도커의 keychain 등록 옵션을 비활성화한 후 다시 실행더니 더이상 오류가 올라오지 않았고 도커 데몬이 정상적으로 시작되었다.  

노란색 박스로 표시한 옵션의 체크박스를 해제하고 다시 실행하면 된다.  
![Keychain](/docker-mac-crash-3.png)

혹시나 해서 [Docker for Mac의 GitHub](https://github.com/docker/for-mac) 이슈를 검색하보니 [#3557](https://github.com/docker/for-mac/issues/3557) 이슈에 누군가가 이미 리포트했다. (~~역시...~~) [lbondarenko 유저의 코멘트](https://github.com/docker/for-mac/issues/3557#issuecomment-502324468)를 참고.