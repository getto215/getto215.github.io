---
title: "Cassandra 소개"
tags:
  - cassandra
---

* NoSQL
* 구글의 빅테이블(Bigtable) + 아마존의 다이나모(Dynamo)
* Client 연결 : 1.0에서는 Thrift, 2.0이후는 CQL
* Admin : CLI, JMX 콘솔

## 데이터 모델
* 정렬된 해시맵(HashMap)
* row를 키-값 형식으로 저장
* 컬럼(column)은 가장 작은 원자 단위 모델
* 단일 로우에 여러 컬럼을 포함할 수 있음
* 컬럼 : 각 엘리먼트는 이름, 값, 타임스탬프 같은 페어로 구성 (슈퍼컬럼: 더 큰 논리 단위)
* 모든 데이터의 페어는 클라이언트와 타임스탬프에서 가져옴
* 컬럼 패밀리 : 컬럼 그룹핑. 테이블과 동일. 더 높은 논리 단위 (슈퍼컬럼패밀리 : 더 큰 논리 단위)
* 키 스페이스(Key space) : 관계형 스키마와 동일
* 복제 계수(replication factor)는 키 스페이스마다 고유

![alt http://a4academics.com/tutorials/83-hadoop/846-cassandra](/assets/img/2018-05-03-Data-Modeling-Cassandra-Architecture.png "Data Modeling Cassandra Architecture")

![alt http://a4academics.com/tutorials/83-hadoop/846-cassandra](/assets/img/2018-05-03-Column-family-Cassandra-architecture.png "Column family Cassandra architecture")

![alt https://www.slideshare.net/yellow7/cassandralesson-datamodelandcql3](/assets/img/2018-05-03-apache-cassandra-lesson-data-modelling-and-cql3-11-638.jpg "RDBMS vs. Cassandra data design")

## 데이터 스토리지
* 카산드라는 짧은 시간에 많은 양의 데이터를 처리하도록 설계됨
* Commit log: 지속 가능성(tolerance?)을 보장할 수 있도록 새로운 데이터가 모두 저장
* 데이터가 커밋 로그 파일에 성공적으로 저장되면 가장 최신 데이터의 기록은 멤테이블(memtable)이라는 메모리 구조에 저장됨 (커밋로그와 멤테이블에 동일한 정보가 있으면 쓰기 실패로 간주)
* 맴테이블 내부 데이터는 로우 키(Row key)로 정렬
* 맴테이블이 가득 차면 내용이 SSTable(Sorted String Table)이라는 구조로 하드에 복사(플러시(flush))
* 플러시는 nodetool flush 커맨드를 통해 주기적으로 수행. 수동 실행 가능
* SSTable은 고정되고 정렬된 로우와 값 키의 정렬된 맵을 제공
* 하나의 SSTable에 입력된 데이터는 변경할 수 없으나, 새 데이터 입력은 가능
* SSTable은 64KB의 블록으로 구성(default)
* 블룸 필터(Bloom filter): 연결 절차 최적화를 위한 메모리 구조. 읽기 요청 시 데이터를 메모리에서 먼저 찾도록 함
* 컴팩션(compaction): SSTable의 조각화(row fragmentation)을 줄이기 위해 백그라운드에서 하나의 SSTable로 합치는 작업