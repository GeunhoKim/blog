---
author: "Kim, Geunho"
date: 2019-08-05
draft: true
title: "Hadoop YARN"
---


# Architecture 
YARN에 대해서 얘기하기 전에, 먼저 하둡 1에서 Job의 관리가 어떻게 이루어졌는지 살펴보겠다.  
[NameNode](/posts/hadoop-namenode/)와 [DataNode](/posts/hadoop-datanode/)가 하둡의 분산 파일 시스템인 HDFS를 구성하는 주요 컴포넌트