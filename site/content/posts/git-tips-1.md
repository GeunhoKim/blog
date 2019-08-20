---
author: "Kim, Geunho"
date: 2019-08-20
title: "Git Tips - 과거 커밋으로부터 텍스트 검색하기"
---


`git log`은 커밋 히스토리를 조회하는 명령어이다. git repository에서 명령어를 입력하면 커밋의 내용을 시간의 역순으로 보여준다.

```bash
git log

commit d5acbda550c5877dc957158229acf7a76044118a (HEAD -> master, origin/master, origin/HEAD)
Author: GeunhoKim <geunho.khim@gmail.com>
Date:   Mon Aug 19 23:18:41 2019 +0900

    Migrate posts from geunhokhim.wordpress.com

commit 26c538203ece30ea4179735d2a41af15b6a01ce7                                                                                                                                                                 Author: GeunhoKim <geunho.khim@gmail.com>
Date:   Mon Aug 19 00:48:12 2019 +0900

    Publish a post: hadoop yarn
...
```

커밋 아이디와 저자, 커밋일시 그리고 커밋 메시지까지 일목요연하게 보여주는데, `-S` 명령어를 사용해서 커밋 히스토리로부터 문자열을 검색할 수 있다.  
이미 삭제되어 찾을 수 없는 문자열을 찾을 때 유용하다.  

```bash
# 문자열에 공백이 포함되지 않은 경우 따옴표는 제외해도 됨
git log -S"검색할 문자열" -p
```
`-p` 옵션에 의해 찾은 파일의 diff 값까지 보여주는데 예제를 살펴보자.

```bash
git log -S리눅스 -p

commit b73e05591ea08f5c0b562d4024a06d067bd8e33c
Author: GeunhoKim <geunho.khim@gmail.com>
Date:   Sun Aug 11 23:24:49 2019 +0900

    Migrate posts from geunhokhm.wordpress.com
diff --git a/site/content/posts/filesystem-permission.md b/site/content/posts/filesystem-permission.md
new file mode 100644
index 0000000..224ec83
--- /dev/null
+++ b/site/content/posts/filesystem-permission.md
@@ -0,0 +1,106 @@
+---
+author: "Kim, Geunho"
+date: 2016-04-15
+draft: true
+math: true
+title: "Linux 파일 시스템 접근 권한"
...
```

커밋 히스토리로부터 `리눅스` 키워드를 검색해서 diff까지 출력했다. 위 출력 가장 마지막 줄에 단어가 걸린 것을 확인할 수 있다.  
만약 해당하는 커밋 아이디만 검색하면 된다면 `--pretty=oneline` 옵션을 넣는다.

```bash
git log -S리눅스 --pretty=oneline

b73e05591ea08f5c0b562d4024a06d067bd8e33c Migrate posts from geunhokhm.wordpress.com
```
커밋 아이디와 메시지만 출력되면서 하나의 커밋이 한 줄로만 출력되도록 할 수 있다.

마지막으로 특정 파일 경로, 혹은 특정 확장자만 검색하고 싶다면 어떻게 할까?  
명령어 마지막에 `-- *.확장자`를 추가하거나, `-- /파일의/상대/경로`를 추가하면 된다. `--` 다음에 공백으로 구분하는 것을 주의한다.  

