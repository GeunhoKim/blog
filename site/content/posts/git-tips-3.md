---
author: "Kim, Geunho"
date: 2020-05-31
title: "Git Tips - Sparse checkout 하기"
---

프로젝트 규모가 큰 repository나 mono repository로 구성한 repository는 개발 도구에서 프로젝트를 불러오면 빌드 시간이 오래걸릴 뿐만 아니라 개발 장비의 리소스도 많이 차지하게 된다.
git의 [core.sparseCheckout](https://git-scm.com/docs/git-read-tree#_sparse_checkout) 옵션을 활성화 하거나, [sparse-checkout](https://git-scm.com/docs/git-sparse-checkout) 명령어를 사용하면 이런 경우를 해결할 수 있다.

# core.sparseCheckout 옵션
먼저 옵션을 활성화한다.
```
git config core.sparseCheckout true
```

그리고 체크아웃할 파일/폴더 혹은 제외할 대상을 아래 파일에 설정하면 끝.
다음 예제에서는 README.md 파일을 제외한 모든 파일을 체크하도록 설정한다.
```bash
touch .git/info/sparse-checkout

# 파일 편집기로 열어서 다음과 같은 내용을 추가
# 
# 대상에서 제외하려면 !를 붙여준다.
/* 
!README.md
```

# sparse-checkout 명령어
위의 일련의 작업을 명령어를 통해 설정할 수 있도록 제공되는 도구이다. 
동일한 작업을 다음과 같이 진행할 수 있다.
```bash
# core.sparseCheckout 옵션 활성화
# 
# --cone 옵션을 추가하면 root 디렉토리의 모든 파일을 체크아웃 한다.
git sparse-checkout init 

# 공백응로 구분하여 대상을 설정한다.
git sparse-checkout set /* !README.md
```


