---
author: "Kim, Geunho"
date: 2019-08-28
title: "Mac Terminal에서 텍스트 편집기 바로 열기"
---

Mac의 Terminal에서 이런 저런 작업을 하다가 파일을 수정할때 `vim`과 같은 텍스트 편집기를 사용한다.  
가끔은 자주 사용하는 GUI 텍스트 편집기로 파일을 수정하고 싶을 때 곧바로 Terminal에서 어떻게 명령어를 입력해야 할까? 답은 `open` 명령어에 있다. 

```
man open

OPEN(1)                   BSD General Commands Manual                  OPEN(1)

NAME
     open -- open files and directories

SYNOPSIS
     open [-e] [-t] [-f] [-F] [-W] [-R] [-n] [-g] [-j] [-h] [-s sdk] [-b bundle_identifier] [-a application] file ... [--args arg1 ...]

DESCRIPTION
     The open command opens a file (or a directory or URL), just as if you had double-clicked the file's icon. If no application name is specified, the default application as determined via LaunchServices is used to open the specified files.

     If the file is in the form of a URL, the file will be opened as a URL.

     You can specify one or more file names (or pathnames), which are interpreted relative to the shell or Terminal window's current working directory. For example, the following command would open all Word files in the current working direc-
     tory:

     open *.doc

     Opened applications inherit environment variables just as if you had launched the application directly through its full path.  This behavior was also present in Tiger.

     The options are as follows:

     -a application
         Specifies the application to use for opening the file

...
```

`open` 명령어는 파일을 여는 명령어인데, `-a` 옵션을 사용하면 어떤 애플리케이션으로 파일을 열것인지 명시할 수 있다.  

```
open -a "application name" /path/to/file
```

`application name`에는 실행할 편집기 이름을 넣어준다. `/Applications`에 설치되어 있는 이름 그대로 넣으면 된다.  
예를 들어 [Visual Studio Code](https://code.visualstudio.com)로 폴더를 열고 싶다면,
```
open -a "Visual Studio Code" ~/Downloads
```

처럼 입력하면 곧바로 애플리케이션을 실행하면서 해당 폴더를 연다. 이 명령어를 다시 `alias`로 등록하거나, 애플리케이션의 바이너리 파일을 `PATH` 환경 변수에 직접 등록해서 사용하면 쉽게 명령어를 입력할 수 있다.(후자의 경우 `open` 명령어를 사용하지 않아도 된다)  
나의 경우 명령어 이름을 입맛에 맞게 지정할 수 있는 `alias` 방법을 선호한다. 
```bash
# profile 편집
vim ~/.profile

# 아래 라인 추가하고 저장
alias vs='open -a "Visual Studio Code"'

# profile 다시 로드
source ~/.profile

# 실행
vs ~/Downloads
```

짧은 명령어로 GUI 편집기를 실행하는 모습을 볼 수 있다.
