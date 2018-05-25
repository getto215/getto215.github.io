---
title: "Kafka 구성과 주요 설정"
tags:
  - kafka
  - zookeeper
---

빅데이터 플랫폼에서 주로 메시지 큐, 버퍼 역할을 하는 Kafka 구성과 주요 설정에 대해 0.10 이후 버전을 기준으로 정리하였습니다. 기본 설정만으로도 운영하는데 큰 이슈는 없으나 production에서는 살펴봐야할 설정들을 다뤘습니다. 

## 설계
* 마스터가 없음(master-less)
* 메타 데이터(metadata)
  * Kafka의 메세지 상태는 consumer 레벨에서 유지 관리
  * 동일한 메시지를 여러번 배달되거나 오류로 인한 메세지 손실을 방지함
* Push와 Pull
  * producer는 메세지를 broker에 보내고 consumer는 메세지를 broker로부터 가져옴
* 보존(retention)
  * 재구독 허용
* 저장소(storage)
  * 파일 시스템에 캐싱 저장
  * 디스크 캐싱과 플러시를 설정할 수 있음
* 동기식(synchronous)
  * producer는 메세지를 브로커에 보낼 때 동기식 또는 비동기식 옵션을 제공

--------

## Kafka Broker

#### Broker config 
server.properties
  * offset.topic.. : offset 토픽관련 옵션
  * compression.type : 토픽의 최종 압축 형태, producer는 producer가 보내는 압축 형태를 유지하라는 옵션 (default: producer)
  * message.max.bytes : 카프카에서 허용하는 가장 큰 메세지 크기 (int, default: 1000012, 1MB 안됨)
  * unclean.leader.election.enable : ISR 그룹에 포함되지 않은 마지막 리플리카를 리더로 인정 (default: true)
  * log.flush.interval.ms : 메시지가 디스크로 플러시 되기 전 메모리에 유지하는 시간(default:null, null일 경우 log.flush.scheduler.interval.ms값)
  * log.flush.interval.messages : 메시지가 디스크로 플러시 되기 전 누적 메시지 수


#### 디자인
페이지 캐시 활용
  * sata 디스크 사용해도 무방
  * heap은 보통 5G면 충분, 남은 메모리는 캐시로 활용. 하나의 머신에 다른 app과 함께 쓰는건 권장하지 않음

배치 전송 처리
  * 작은 메시지를 묶어서 처리. 네트워크 오버헤드 줄임



#### 파티션 관리

무조건 파티션 수를 늘리면 안되는 이유
  * 파일 핸들러의 낭비 : 파티션 수가 많을수록 파일 핸들 수 역시 증가
  * 장애 복구 시간 증가 : 일부 브로커가 다운되었을 경우 각 파티션별로 리더 선출 작업을 하게 되는데 파티션이 많을 경우 복구가 지연됨

적절한 파티션 수는
  * 목표 처리량을 기준으로 함
  * 프로듀서, 컨슈머 모두 고려해서 max로 파티션을 할당함
  * 브로커 당 2,000개를 max 파티션 수로 권장


#### 리플리케이션
* 리더와 팔로워로 역할이 나눠있음. 가장 중요한건 모든 읽기, 쓰기가 리더를 통해서만 이뤄짐
* ISR(In Sync Replica)의 구성원이여야 리더가 될 수 있음
* replica.lag.time.max.ms : 리더는 팔로워들이 주기적으로 데이터를 확인하는지 체크. 이상이 발견될 경우 ISR 그룹에서 제거됨. (default:10000)

#### 세그먼트 파일(segment file)
* 내부적으로 모든 파티션은 동일한 크기의 세그먼트 파일 세트로 표현되는 논리 로그 파일로 구성되어 있음
* 메세지가 특정 수에 도달하면 세그먼트 파일이 디스크에 플러시
* 파일이 플러시되면 consumer가 해당 메세지를 사용할 수 있음

#### 오프셋(offset)
* 고유 순차 번호
* 파티션 내부의 메세지를 식별하는데 사용


#### 리더(leader)
* 각 파티션마다 리더인 하나의 kafka 서버가 있음
* 구성원(follower)은 나머지 서버
* 리더는 해당 파티션에 대한 읽기 요청과 쓰기 요청을 조정
* 구성원 서버는 리더 서버의 데이터를 비동기적으로 복제. 리더가 실패하면 나머지 서버 중 하나가 새로운 리더


#### 로그 컴팩션(log compaction)
* 시간 기반 메세지 보존
* 보존 정책 종류 : Coarse-grained(시간 기반), Fine-grained(메세지 기반)
* Kafka 보존 정책 : 토픽별(시간 기반), 크기 기반, 로그 컴팩션 기반
* 로그 컴팩션 보장
  * 읽기가 오프셋 0부터 시작
  * 메세지는 순차적인 오프셋을 가지고 해당 오프셋은 결코 변경되지 않음
  * 메세지 순서는 항상 보존
  * 백그라운드 스레드 그룹은 로그 세그먼트 파일을 다시 복사. 로그 앞부분에 키가 나타난 레코드는 삭제


#### 메세지 압축
* 네트워크 오버헤드는 감소, 브로커 CPU 사용율(리더)은 증가
* 메세지 그룹을 압축
* 메세지 그룹이 압축되어 단일 메세지로 consumer에 제공
* 0.8.0 이전에는 consumer가 해당 메세지를 압축 해제했으나 이후 버전에서는 브로커가 처리
* 데이터 센터 간 미러링 할 경우 사용할만함

#### 복제
* 복제는 메세지가 발행되고 사용, producer와 consumer 모두 복제를 알고 있음
* 하나의 복제본은 리더 역할을 함. 주키퍼는 복제본의 리더를 알고 있음
* 비동기 복제 : 리더에 저장되면 producer에 ack를 보냄. 빠르나 내결함성이 없음
* 동기 복제 : 모든 구성원에서 복제본에 대한 ack를 받을 때까지 기다린 후 producer에 ack를 보냄. 느리지만 안정적

-------

## Kafka Producer
비동기 전송
  * org.apache.kafka.clients.producer.Callback 클래스 사용
  * kafka 브로커에 응답을 기다리지 않고 처리

python kafka github
  * kafka-python : 주로 사용됨
  * confluent-kafka-python : 성능은 빠름. librdkafka(C lib)이 필요


#### Producer config
* ack (default:1)
  * 0: 프로듀서는 서버로부터 어떠한 ack도 기다리지 않음. 유실율 높으나 높은 처리량
  * 1: 리더는 데이터를 기록, 모든 팔로워는 확인하지 않음
  * -1(또는 all): 모든 ISR 확인. 무손실
* buffer.memory: 프로듀서가 브로커로 데이터를 보내기 위해 잠시 대기하는 메모리량(bytes, default: 33554432, 32MB)
* batch.size: 같은 파티션으로 보내는 여러 데이터를 함께 배치로 보내기 위한 사이즈. 정의된 크기보다 큰 데이터는 배치를 시도하지 않음. 고가용성이 필요할 경우 배치 사이즈를 지정하지 않음 (bytes, default: 16384, 16KB)
* linger.ms: 배치형태의 메시지를 보내기 전에 추가적인 메시지들을 위해 기다리는 시간을 조정. 0보다 큰 값을 설정하면 지연은 발생하지만 처리량은 좋아짐 (default:0, 지연 없음)
* max.request.size: 프로듀서가 보낼 수 있는 최대 메시지 사이즈. (default:  1MB)

-------

## Kafka Consumer
New와 Old 컨슈머 차이 : old는 오프셋을 zk에 저장

#### Consumer config
* fetch.min.bytes : 한번에 가져올 수 있는 최소 데이터 사이즈. 만약 지정한 사이즈보다 작은 경우 요청에 응답하지 않고 데이터가 누적될 때까지 기다림 (default: 1)
* auto.offset.reset: 카프카 초기 오프셋이 없거나 현재 오프셋이 더 이상 존재하지 않을 경우(데이터가 삭제)에 다음 옵션으로 리셋함
  * earliest, lastest, none (default: latest)
* fetch.max.bytes : 한 번에 가져올 수 있는 최대 데이터 사이즈 (default: 52428800, 50MB)
* request.timeout.ms : 요청에 대해 응답을 기다리는 최대 시간 (default: 305000)
* session.timeout.ms : 컨슈머와 브로커 사이의 세션 타임 아웃 시간(default: 10초). 타임아웃되면 해당 컨슈머는 종료되거나 장애로 판단하고 리밸런스를 시도함. heartbeat.interval.ms(기본 3초)와 밀접한 관련이 있음. GC를 고려하여 적당한 시간 조정 필요
* max.poll.records: 단일 호출 poll()에 대해 최대 레코드 수를 조정. 이 옵션을 통해 app이 폴링 루프에서 데이터 양을 조정할 수 있음 (default: 500)
* max.poll.interval.ms: 하트비트를 통해 살아는 있으나 실제 메세지를 가져가지 않을 경우. 주기적으로 poll을 호출하지 않으면 장애라고 판단하고 컨슈머 그룹에서 제외 (default: 300,000)
* fetch.max.wait.ms: fetch.min.bytes에 의해 설정된 데이터보다 적은 경우 요청에 응답을 기다리는 최대 시간 (default: 500)

#### 커밋과 오프셋
* __consumer_offsets에 오프셋 정보를 저장(0.9 이후)
* 자동 커밋
  * enable.auto.commit=true : 5초다마 컨슈머는 poll()을 호출할 때 가장 마지막 오프셋을 커밋
  * auto.commit.interval.ms 로 시간 조정 가능
  * commit전 리밸런싱(컨슈머 삭제, 다운 등)이 일어나면.. 중복 소비됨
* 수동 커밋: db등에 전송이 완료된 후 명령어 실행으로 처리 가능. consumer.commitSync();
* 특정 파티션 할당
~~~java
String topic = "some";
TopicPartition p1 = new TopicPartition(topic, 0);
consumer.assign(Array.asList(p1));
while (true){
    ConsumerRecords<String,String> records = consumer.poll(100);
    for (ConsumerRecord<String, String>) record : records){
        System.out.printf("%s, %s, %d, %s, %s",
            record.topic(),
            record.partition(),
            record.offset(),
            record.key(),
            record.value()
        )
    }
     consumer.CommitSync();
     consumer.close();
}
~~~
* 특정 오프셋으로부터 메시지 가져오기
  * seek()를 사용
~~~java
TopicPartition p1 = new TopicPartition(topic, 0);
consumer.assign(Array.asList(p1));
consumer.seek(p1,2); // 2번 오프셋부터 가져오기
while (true){
    ConsumerRecords<String,String> records = consumer.poll(100);
    ...
}
~~~

-------

## Kafka 운영
* zookeeper 스케일 아웃
  * 3-->5대 : 60,000개 요청 수를 더 처리할 수 있음
  * 가급적 리더를 가장 마지막에 재시작

#### kafka 모니터링
* JMX 설정
~~~bash
# kafka-server-start.sh
export JMX_PORT=19999
...
~~~

-------

## Kafka Streams API
Stateful : 상태기반. 이전 스트림을 처리한 결과를 참조해야할 경우 (단어 빈도수 세기, 실시간 추천 프로그램)

특징
  * 시스템이나 카프카에 대한 의존성 없음
  * 이중화된 로컬 상태 저장소 지원
  * exactly-once
  * 밀리초 단위의 처리 지연을 보장하기 위해 한 번에 한 레코드만 처리
  * 고수준 스트림 DSL(Domain Specific Language)를 지원, 저수준 프로세싱 API도 지원
  * 별도의 스트리밍 엔진을 사용하지 않고도 간단하게 실시간 분석을 수행할 때

카프카 스트림은 스트림 처리를 하는 프로세서들이 서로 연결되어 형상(토폴로지)을 만들어 처리하는 API
  * 소스 프로세서, 스트림 프로세서, 스트림, 싱크 프로세서...


Hello Kafka Streams (http://kafka.apache.org/documentation/streams/)
~~~scala
import java.lang.Long
import java.util.Properties
import java.util.concurrent.TimeUnit
 
import org.apache.kafka.common.serialization._
import org.apache.kafka.common.utils.Bytes
import org.apache.kafka.streams._
import org.apache.kafka.streams.kstream.{KStream, KTable, Materialized, Produced}
import org.apache.kafka.streams.state.KeyValueStore
 
import scala.collection.JavaConverters.asJavaIterableConverter
 
object WordCountApplication {
 
    def main(args: Array[String]) {
        val config: Properties = {
            val p = new Properties()
            p.put(StreamsConfig.APPLICATION_ID_CONFIG, "wordcount-application")
            p.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-broker1:9092")
            p.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass)
            p.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass)
            p
        }
 
        val builder: StreamsBuilder = new StreamsBuilder()
        val textLines: KStream[String, String] = builder.stream("TextLinesTopic")
        val wordCounts: KTable[String, Long] = textLines
            .flatMapValues(textLine => textLine.toLowerCase.split("\\W+").toIterable.asJava)
            .groupBy((_, word) => word)
            .count(Materialized.as("counts-store").asInstanceOf[Materialized[String, Long, KeyValueStore[Bytes, Array[Byte]]]])
        wordCounts.toStream().to("WordsWithCountsTopic", Produced.`with`(Serdes.String(), Serdes.Long()))
 
        val streams: KafkaStreams = new KafkaStreams(builder.build(), config)
        streams.start()
 
        Runtime.getRuntime.addShutdownHook(new Thread(() => {
            streams.close(10, TimeUnit.SECONDS)
        }))
    }
 
}
~~~

-------

## KSQL을 이용한 스트리밍 처리
저장 기간에 관계없이 스트리밍과 배치 처리를 동시에 실행할 수 있음. 람다 아키텍쳐

> 람다 아키텍쳐 레이어
> * batch layer: raw 데이터가 저장되어 있고, batch 처리하여 배치 뷰 생성
> * serving layer: batch로 분석된 데이터가 저장되어 있고 batch 외에는 쓰기가 안됨
> * speed layer: 실시간 데이터를 집계
> 
> 출처: http://gyrfalcon.tistory.com/entry/람다-아키텍처-Lambda-Architecture

카파 아키텍처(Kappa Architecture) : 데이터의 크기나 기간에 관계 없이 하나의 계산 프로그램을 사용하는 방식
  * 장기 데이터 조회가 필요할 경우 미리 장기 데이터를 따로 저장하는게 아닌 `계산`해서 결과를 그때 전달

#### KSQL 아키텍처
KSQL 서버
  * REST API 서버 : 사용자로부터 쿼리를 받을 수 있는 REST API 서버
  * KSQL 엔진 : 사용자 쿼리를 논리적/물리적 실행 계획으로 변환하고, 지정된 카프카 클러스터의 토픽으로부터 데이터를 읽거나 토픽을 생성해 데이터를 생성하는 역할

KSQL 셸 클라이언트 : KSQL에 연결하고, 사용자가 SQL쿼리문을 작성할 수 있게 함

-------

## Zookeeper
zookeeper 노드 수에 따른 성능
![alt "zookeeper perf"](/assets/img/2018-05-14-zkperfRW-3.2.jpg)


zookeeper를 랙별로 배치
환경설정
  * tickTime : 주키퍼가 사용하는 시간에 대한 기본 측정 단위(ms),  heartbeats , timeouts (default: 2000)
  * initLimit : 팔로워와 리더가 초기에 연결하는 시간에 대한 타임 아웃 tick의 수 (default: 10)
  * syncLimit : 팔로워가 리더와 동기화 하는 시간에 대한 타임 아웃 tick의 수 (zk에 저장된 데이터가 크면 늘려야함, default: 5)
  * dataDir
  * clientPort
  * server.x : 클러스터 구성, 2888과 3888는 노드간 포트

가급적 zk와 kafka는 따로 구성

zk를 사용하는 java app 주의 사항
  * java 기반 app은 full gc가 발생하면 gc타임 동안 일시적으로 멈춤 상태가 됨
  * zookeeper.session.timeout이 너무 짧으면 노드가 다운된 것으로 판단할 수 있음
  * gc타임을 주기적으로 체크, timeout도 3초 이상으로 설정할 필요


systemd을 이용한 init proc(https://lunatine.net/2014/10/21/about-systemd/)

---------

## 참고자료
* [Confluent Kafka 3.2.2](https://docs.confluent.io/3.2.2/kafka/deployment.html#important-configuration-options)
* [Kafka 0.10.2 Docs](http://kafka.apache.org/0102/documentation.html#brokerconfigs)
* [[Book] 카프카, 데이터 플랫폼의 최강자](http://www.kyobobook.co.kr/product/detailViewKor.laf?barcode=9791196203726)
* [[Book] SMACK 스택을 이용한 빠른 데이터 처리 시스템](http://www.kyobobook.co.kr/product/detailViewKor.laf?ejkGb=KOR&mallGb=KOR&barcode=9791161750828&orderClick=LAG&Kc=)
* [kafka 브로커 설정](https://free-strings.blogspot.kr/2016/04/blog-post.html)