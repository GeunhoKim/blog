---
author: "Kim, Geunho"
date: 2019-08-20
title: "Git Tips - 로컬 커밋 복구하기"
---


`git reflog`는 로컬 git repository의 HEAD 변경 이력을 보여주는 명령어이다.  

```bash
git reflog

6552006 (HEAD -> master, origin/master, origin/HEAD) HEAD@{0}: commit: Publish a post about git tips
d5acbda HEAD@{1}: reset: moving to HEAD   
d5acbda HEAD@{2}: reset: moving to origin/master                                                                                                                                                                d5acbda HEAD@{3}: reset: moving to HEAD                                                                                                                                                                         d5acbda HEAD@{4}: reset: moving to HEAD                                                                                                                                                                         d5acbda HEAD@{5}: pull: Fast-forward                                                                                                                                                                      d5acbda HEAD@{2}: reset: moving to origin/master                                                                                                                                                                d5acbda HEAD@{3}: reset: moving to HEAD                                                                                                                                                                         d5acbda HEAD@{4}: reset: moving to HEAD                                                                                                                                                                         d5acbda HEAD@{5}: pull: Fast-forward                                                                                                                                                                            26c5382 HEAD@{6}: pull: Fast-forward  
...
```

로컬에 커밋을 하면 HEAD를 과거로 reset 하더라도 이렇게 모두 기록이 남아있다. 실수로 커밋을 날려버렸다면 당황하지 말고 다음과 같은 명령어로 복구한다.

```bash
# 먼저 HEAD 이력을 출력
git reflog

d7250d58 (HEAD -> feature/my-branch) HEAD@{0}: reset: moving to d7250d583c09605f1409c0dc7c35744aaae91cd2
c685b83c HEAD@{1}: rebase -i (finish): returning to refs/heads/feature/my-branch
c685b83c HEAD@{2}: rebase -i (squash): Some comments
51e05171 HEAD@{3}: rebase -i (squash): # This is a combination of 7 commits.
0f46bda8 HEAD@{4}: rebase -i (squash): # This is a combination of 6 commits.

# 0f46bda8 HEAD@{4}로 돌아가자
git reset 'HEAD@{4}'
```

