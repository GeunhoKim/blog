---
author: "Kim, Geunho"
date: 2016-04-15
draft: true
math: true
title: "Linux 파일 시스템 접근 권한"
---


# 파일 시스템의 접근 권한
리눅스를 포함한 POSIX 계열 운영체제에서는 “전통적인 유닉스 권한” (Traditional Unix Permissions)을 따른다. 파일 목록을 보는 명령어를 입력하면 아래와 같은 출력을 볼 수 있다.  

```bash
ls -al
# 접근 권한                  소유자        소유그룹
drwxr‐‐r‐‐        7        root        root    62    Apr    14    12:21    .
drwxrwxr-x        3        root        root    19    Apr    14    19:49    ..
drwxr‐‐r‐‐        1        root        root    19    Apr    14    19:49    bin
drwxr‐‐r‐‐        1        root        root    19    Apr    14    19:49    etc
‐rwxr-xr-x        1        root        root    19    Apr    14    19:49    script.sh
lrwxr‐‐r‐‐        1        root        root    19    Apr    14    21:32    link.sh
```

각 라인의 가장 앞 열 개의 문자열은 파일에 대한 접근 권한을 나타낸다.  

![파일 접근 권한](/filesystem-permission-1.png) _표 1. 파일 접근 권한을 나타내는 열 개의 문자열_

위의 표에서 보여주는 것 처럼 가장 첫 번째 문자는 파일의 형식을 나타낸다. 다음 세 개의 문자는 파일의 소유자에 대한 읽기(Read), 쓰기(Write), 실행(eXecute) 권한을 나타내며, 다음 세 문자는 소유그룹에 대한 권한, 다음은 그 외의 계정에 대한 권한을 나타낸다.  
만약 `-rwxr-xr-x`라고 표기 되어 있다면, `-`로 파일을 뜻하고, 소유자는 모든 권한을, 그룹과 그 외 계정은 읽기 및 실행 권한이 부여된 것이다.  


nodejs 애플리케이션을 통해 웹 서비스를 제공하는 서버가 있다고 가정한다.  
nodejs가 root에 의해 설치가 되면 실행 파일의 기본 권한은 아래와 같다.  

```bash
‐rwx‐r‐xr‐x    1    root    root
```

얼핏 보면 보안상 큰 문제가 없어 보이지만 공격자가 어떤 계정이라도 한번 탈취하게 되면 nodejs의 실행을 마음대로 할 수 있게 된다. 이런 경우 아래와 같은 방법으로 권한을 변경할 수 있다.  

```bash
# root 접속
su –

NODE_PATH=$(which nodejs)

# 실행 파일의 소유자를 nodejs 실행 계정으로 변경
chown nodeapp:nodeapp $NODE_PATH

# 실행 파일의 group 및 others에 대한 접근 권한을 제한 (권한 없음 또는 읽기로)
chmod 744 $NODE_PATH

ls -al $NODE_PATH
‐rwx‐r‐‐r‐‐    1    nodeapp    nodeapp
```

위 예제에서 nodejs 실행 파일에 대한 소유자와 소유권한 설정을 `chown`, `chmod` 명령어를 이용해서 변경했다. 두 명령어에 대해 좀더 알아본다.  


# 파일 소유자 변경: chown (change owner)
위 권한 변경 예제에서 세 번째 줄의 chown은 “change owner”의 줄임말로, 파일 소유자를 변경하는 명령어이다. 소유자 변경은 명령어를 실행하는 계정에 root 권한이 있거나, 같은 그룹에 해당하는 파일에 한에서만 수정할 수 있다.  

```bash
# 단일 파일을 변경
chown $소유자:$소유그룹 $대상파일경로 

# -R recursive 옵션, 상위 폴더를 포함하여 모든 하위 파일들에 대해 변경
chown -R $소유자:$소유그룹 $상위폴더 
```


# 파일 접근 권한 변경: chmod (change mode)
chmod는 파일의 접근 권한을 변경하는 명령어이다.

```bash
chmod $mode $대상파일경로

chmod -R $mode $상위폴더
```

mode에 들어가는 값은 두 가지 방법으로 표현 할 수 있다.

* octal mode: 세 자리 8진수로 표현
* symbolic mode: 변경 대상(user, group, others), 연산자, 그리고 모드 (read, write, execute, …)를 심볼 문자로 표현

symbolic mode는 다음 예를 참고한다.
```bash
# 소유자에 대해 실행 권한 추가
chmod u+x

# 모든 사용자에 대해 읽기 권한 추가
chmod a+r

# 다른 사용자에 대해 쓰기 권한 제거
chmod o-x
```

octal mode는 먼저 아래 표를 살펴본다.
![chmod의 octal mode](/filesystem-permission-2.png) _표 2. chmod의 octal mode_

Read는 4, Write는 2, eXecute는 1로 표현 되는데 이는 8진수 하나를 비트로 표현 했을 때 각 자리수에 해당하는 값임을 알 수 있다.

$$ 2^2 + 2^1 + 2^0 $$

따라서 `rwx`인 경우는 4 + 2 + 1 = 7 이 되고, `‐‐x`인 경우는 0 + 0 + 1 = 1이 된다.   
`rwx r‐‐ r‐‐` 은 7 4 4 가 되는 것이다. (소유자는 읽고 쓰고 실행할 수 있는 권한이 있지만 그 외 소유 그룹과 다른 유저는 읽기만 가능)
