---
author: "Kim, Geunho"
date: 2017-02-02
title: "Ubuntu에서 RPM 파일 설치하기"
---

# RPM에 대해서
RPM은 Red Hat Package Manager의 약자로 원래 레드햇 리눅스에서 사용되던 패키지 매니저와 배포용 패키지 파일을 뜻한다.

현재는 레드햇 리눅스 뿐만 아니라 CentOS, Fedora에서도 널리 사용되고 있다.
이렇게 리눅스의 표준 패키지 포맷 중 하나가 되면서, 배포 파일이 RPM으로만 제공되는 경우가 있는데, 이런 경우 기본적으로 지원되지 않는 Debian 계열 – Ubuntu에서는 어떻게 설치하고 사용할 수 있는지 알아본다.

# Ubuntu에서 RPM 파일 설치하기
Ubuntu에서는 rpm 파일로 곧바로 설치할 수 없다. `apt` 패키지 매니저를 통해 `rpm` 도구를 설치했다 하더라도 아래와 같은 에러가 발생한다:

```
rpm: RPM should not be used directly install RPM packages, use Alien instead!
```

`alien`을 대신 사용하라고 친절하게 설명해준다. `alien`은 에일리언, 이름 그대로 `rpm` 파일을 Debian용 파일인 `deb`로 변환하거나 곧바로 설치할 수 있도록 도와주는 도구이다. 

현재 경로상에 `target_rpm_file.rpm`이라는 파일이 있다고 가정하면, 아래와 같이 설치해서 파일을 변환할 수 있다.
```
# alien 설치
apt-get install alien -y

# 파일 변환
alien -c target_rpm_file.rpm
```

파일 변환 명령어 수행이 완료되면  `target_rpm_file.deb` 파일이 생성된 것을 확인 할 수 있다.(용량에 따라서 시간이 걸릴 수 있다) 이제 `dpkg` 도구를 사용해서 설치하면 된다.
```
dpkg -i target_rpm_file.deb
```

만약 파일 변환 없이 곧바로 설치하고 싶다면 `alien -i` 명령어로 곧바로 설치할 수 있다.
```
alien -i target_rpm_file.rpm
```
