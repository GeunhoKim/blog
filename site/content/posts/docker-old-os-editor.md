---
author: "Kim, Geunho"
date: 2019-11-07
title: "Docker - 오래된 버전의 OS 베이스 이미지에 텍스트 편집기 설치하기"
---

테스트 목적으로 도커 컨테이너를 띄워 작업하다보면 가끔 컨테이너 쉘을 직접 실행해서 파일을 편집하는 경우가 있다.
베이스 이미지 용량을 줄이기 위해 텍스트 편집기 도구는 모두 삭제하고 이미지로 배포하기 때문에 별도로 설치해야한다.

Debian 계열인 경우 다음과 같이 설치하게 되는데,

```
apt-get update
apt-get install vim -y
```

LTS 기간까지 지난 버전인 경우 `apt-get update` 명령 실행시 오류가 발생한다.

```
# Wheezy
apt-get update
Ign http://security.debian.org wheezy/updates Release.gpg
Ign http://security.debian.org wheezy/updates Release
Ign http://deb.debian.org wheezy Release.gpg
Err http://security.debian.org wheezy/updates/main amd64 Packages

...

Err http://deb.debian.org wheezy/main amd64 Packages
  404  Not Found [IP: 151.101.196.204 80]
Err http://deb.debian.org wheezy-updates/main amd64 Packages
  404  Not Found [IP: 151.101.196.204 80]
W: Failed to fetch http://security.debian.org/dists/wheezy/updates/main/binary-amd64/Packages  404  Not Found [IP: 151.101.196.204 80]

W: Failed to fetch http://deb.debian.org/debian/dists/wheezy/main/binary-amd64/Packages  404  Not Found [IP: 151.101.196.204 80]
W: Failed to fetch http://deb.debian.org/debian/dists/wheezy-updates/main/binary-amd64/Packages  404  Not Found [IP: 151.101.196.204 80]

E: Some index files failed to download. They have been ignored, or old ones used instead.
```

Wheezy (Debian 7) 베이스 이미지에서 실행하면 발생하는 오류인데 apt repository를 찾지 못해서 404 오류를 내고 있다.

LTS 기간이 끝나면서 apt repository도 아카이브 저장소로 이동[^1]한 것이 원인으로 해결 방법은 간단한데,
apt repository를 아카이브 저장소로 변경하기만 하면 된다.

```
echo "deb http://archive.debian.org/debian wheezy main" > /etc/apt/sources.list
echo "deb http://archive.debian.org/debian-security wheezy/updates main" >> /etc/apt/sources.list

# 이제 다운로드를 하기 시작한다.
apt-get update

apt-get install vim -y
```

[Debian LTS](https://wiki.debian.org/LTS) 기간을 확인해보면, 8버전인 Jessie도 내년 6월이면 지원이 끝나게 된다.
내년 6월 이후부터 8버전을 사용하는 베이스 이미지에서 비슷한 문제가 발생할 수 있으니 기억해두는 것이 좋겠다.

[^1]: https://www.debian.org/distrib/archive
