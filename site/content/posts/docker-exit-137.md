---
author: "Kim, Geunho"
date: 2020-04-07
title: "Docker 컨테이너 비정상 종료 문제 - non-zero exit (137)"
---


로컬 컨테이너 환경에서 분산 환경 테스트 중 컨테이너 하나가 `exit(137)`로 로그 없이 죽는 현상이 발생했다.

`SIGKILL`에 의해 컨테이너가 종료되면 이와 같은 코드를 남기는데, 다시 살리면 멀쩡하던 다른 컨테이너가 또 종료되는 일이 반복되었다.
137 종료 코드가 발생하는 케이스를 좀더 찾아보니 OOM에 의해서도 발생할 수 있다는 사실을 발견!

https://success.docker.com/article/what-causes-a-container-to-exit-with-code-137

컨테이너는 java 애플리케이션이었는데, 호스트인 로컬 환경의 메모리 부족이 원인이었다!
로컬 환경은 Mac으로, Docker for Mac으로, 가상 머신 위에서 도커 엔진이 동작하는데... 가상 머신 기본 메모리 값이 2GB 밖에 되지 않았던 것.

2GB -> 4GB로 증설해서 문제를 해결했다.

