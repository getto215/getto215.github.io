---
title: "Hadoop HDFS HA 설정"
tags:
  - hadoop
---
http://getto-dev.tistory.com/entry/Hadoop-HDFS-HA


Hadoop HDFS HA를 위한 설정과 구동

### 시스템 구성
* HA구성만을 다루며, Datanode 등은 모두 제외

| Host        | Usage          |
|:------------|:------------------|
| dpm-001     | Namenode(Active), JournalNode, Zookeeper|
| dpm-002     | Namenode(Slave), JournalNode, Zookeeper|
| dpm-003     | JournalNode, Zookeeper|
 
core-site.xml
ssh fence를 사용

Active, Standby 서버간 ssh로 통신할 수 있도록 함. (hdfs계정의keyfile을 사용)

사전에 각 서버에 zookeeper를 설치함

HDFS 접근 시 특정 호스트명이 아닌 hdfs://cluster로 접속