---
title: "Spark 구성요소 소개"
tags:
  - spark
  - yarn
  - hadoop
  - mesos
---

## RDD
Resilient Distributed Dataset. 탄력적인 분산 데이터셋

> 클러스터에 있는 다수의 머신에 분할되어 저장된 읽기 전용 컬렉션. 전형적인 스파크 프로그램은 하나 또는 그 이상의 RDD를 입력받고 일련의 변환 작업을 거쳐 목표 RDD 집합으로 변형된다. '탄력적'이란 단어는 유실된 파티션이 있을 때 스파크가 RDD의 처리 과정을 다시 계산하여 자동으로 복구할 수 있다는 의미

---------

## 스파크 애플리케이션, 잡, 스테이지, 태스크
* MR과 유사하게 잡이라는 개념이 있음
* 단일 맵리듀스의 잡 : 하나의 맵과 하나의 리듀스로 구성
* 스파크의 잡: 임의의 방향성 비순환 그래프(Directed acyclic graph. DAG)인 스테이지(stage)로 구성. 스파크가 실행될 때 스테이지는 다수의 태스크로 분할되고, 각 태스크는 맵리듀스의 테스크와 같이 클러스터의 분산된 RDD 파티션에서 병렬로 실행
* 잡은 항상 RDD 및 공유변수를 제공하는 어플리케이션(SparkContext 인스턴스로 표현되는)의 콘텍스트 내에서 실행됨. 동일한 어플리케이션에서 수행된 이전 잡에서 캐싱된 RDD에 접근할 수 있는 메커니즘을 제공

---------


## 탄력적인 분산 데이터셋 RDD
#### 생성
(1) 객체의 인메모리 컬렉션(병렬)으로 생성
  * 적은 양의 입력 데이터를 병렬로 처리하는 CPU 부하 중심의 계산에 유용
  ~~~scala
  // 병렬 수준 제지정
  sc.parallelize(1 to 10, 10)
  ~~~

(2) HDFS와 같은 기존 외부 저장소의 데이터셋을 사용
  * 스파크에서 파일을 스플릿하는 방식은 하둡과 동일. 따라서 HDFS의 경우 HDFS 블록당 스파크 파티션도 하나.
  ~~~scala
  // 스플릿 수 변경
  sc.textFile(inputPath, 10)
  ~~~

(3) 기존 RDD를 변환

#### 트랜스포메이션과 액션
* 스파크 RDD에 트랜스포메이션과 액션이라는 두 종류의 연산자를 제공
* 트랜스포메이션은 기존 RDD에서 새로운 RDD를 생성. 즉시 실행X (map). 매핑, 그룹화, 집계, 재분할, 샘플링, RDD조인 및 RDD 집합 처리
* 액션은 특정 RDD를 계산하여 어떤 결과를 만들어냄. 결과는 보여지거나 외부에 저장. 즉시 실행 (foreach). RDD를 컬랙션으로 실체화, RDD 통계 계산, RDD에서 특정 개수의 항목을 샘플링, RDD를 외부 저장
* 트랜스포메이션인지 액션이지 파악하는 방법: 반환 타입을 확인. RDD면 트랜스포메이션. 아니면 액션
* === 는 assert 구문에서 사용. == 보다 더 유용한 오류 메세지 제공
~~~scala
assert(sum.collect().toSet === Set(("a", 9),("b",7)))
~~~

#### 지속성
cache를 호출해도 RDD를 메모리에 즉시 캐싱하지 않는다. 대신 스파크 잡이 실행될 때 해당 RDD를 캐싱해야한다고 플래그로 표시해둔다.
~~~scala
scala> tuple.cache()
res1: tuples.type = MappedRDD[4] at map at <console>: 18

// RDD 파티션이 메모리에 보관. 2개 파티션
scala> tuple.reduceByKey((a, b) => Math.max(a, b)).foreach(println(_))
INFO BlockMAnagerInfo: Added rdd_4_0 in memory on 192.168.1.90:64640
INFO BlockMAnagerInfo: Added rdd_4_1 in memory on 192.168.1.90:64640
...
(1950,22)
...
~~~

캐싱된 데이터셋에 다른 잡을 실행하면 RDD가 메모리에 로드된 것을 볼 수 있음
~~~scala
scala> tuple.reduceByKey((a, b) => Math.max(a, b)).foreach(println(_))
INFO BlockManager: Found block rdd_4_0 locally
INFO BlockManager: Found block rdd_4_1 locally
...
~~~

cache()를 호출하면 익스큐터의 메모리에 각 RDD 파티션을 보존한다.

StorageLevel
  * Memory_only: 기본 값. 일반적인 인메모리 객체 표현
  * Memory_only_ser
    * 파티션의 요소를 바이트 배열로 직렬화하여 객체를 압축된 형태로 저장
    * Memory_only보다 더 많은 CPU를 사용
    * 충분한 메모리는 없지만 최종 직렬화 RDD 파티션을 저장하기에 적합하다면 시도
    * 각 RDD를 큰 객체가 아닌 하나의 바이트 배열로 저장하기 떄문에 GC 부담을 줄일 수 있음
  * Memory_and_disk_ser: 메모리에 먼저 저장하고 충분하지 않으면 디스크에 저장
* 파티션을 클러스터에 있는 단일 노드가 아닌 여러 노드에 복제해서 저장하거나 힙메모리를 사용하는 지속성 수준도 있음

#### 직렬화
데이터 직렬화
  * 대부분 크라이오(Kyro)를 사용(자바 직렬화는 효율 떨어짐)

  ~~~scala
  conf.set("spark.serializer", "org.apache.spark.serializer.KyroSerializer")
  ~~~

  * Kyro를 사용할 떄 java.io.Serializable과 같은 특정 인터페이스를 구현한 클래스를 만들지 않아됨. 기존 자바 객체를 RDD에 사용할 수 있음

함수 직렬화
  * 스칼라에서 함수는 표준 자바 직렬화 메커니즘으로 직렬화. 스파크는 원격 익스큐터 노드에 함수를 전송할 때 이를 사용

---------

## 공유변수
#### 브로드캐스트 변수
* 브로드캐스트 변수는 직렬화된 후 각 익스큐터에 전송되며, 나중에 태스크가 필요할 때 언제든지 접근할 수 있도록 캐싱된다.
* 브로드캐스트 변수는 태스크마다 한 번씩 네트워크를 통해 전달되는 클로저의 일부로 직렬화되는 일반적인 변수와는 다르다. 스파크 메모리가 충분하면 모두 메모리에 저장, 부족하면 디스크

~~~scala
val lookup: Broadcast[Map[Int, String]] = sc.broadcast(Map(1->"a",2->"e",3->"i"))
// RDD map에서 변수에 접근하기 위해서는 브로드캐스트 변수의 value() 메서드를 호출해야함
val result = sc.parallelize(Array(2,1,3)).map(lookup.value(__))
assert(result.collect().toSet === Set("a","e","i"))
~~~

* 브로그캐스트 변수는 드라이버에서 태스크로 단방향 전송됨. 브로드캐스트 변수의 값을 변경하거나 드라이버로 역전파할 수 없음. (어큐뮬레이터 사용)

#### 어큐뮬레이터
* 테스크에서 그 값을 증가만 시킬 수 있는 공유변수


---------


## 스파크 잡 수행 분석
* 드라이버: SparkContext를 포함한 애플리케이션을 관리하고 잡의 태스크를 스케줄링한다. 일반적으로 클러스터 매니저가 관리하지 않는 클라이언트에서 실행
* 익스큐터: 애플리케이션과는 분리되어 있으며, 애플리케이션의 실행을 관장하고 애플리케이션의 태스크를 실행한다. 클러스터에 있는 머신에서 실행(항상 그런건 아님)

#### 잡 제출
![Alt "cluster-overview"](/assets/img/2018-05-24-spark-cluster-overview.png)
![Alt "spark job execution"](/assets/img/2018-05-24-spark-Internals-of-job-execution-in-spark.jpg)
![Alt "execution flow"](/assets/img/2018-05-24-spark-apache-spark-in-depth-core-concepts-architecture-internals-10-638.jpg)

* 드라이버
  1. 스파트 잡은 RDD에 count()와 같은 액션이 호출될 때 자동으로 제출 --> SparkContext
  2. SparkContext --(Job실행)--> DAG 스케줄러
  3. DAG 스케줄러 --(스테이지 제출)--> 테스크 스케줄러
  4. 테스크 스케줄러 --(테스크 구동)--> 스케줄러 백앤드
  5. 스케줄러 백앤드 --(익스큐터 테스크 구동)--> 익스큐터 백앤드
* 익스큐터
  5. 익스큐터 백앤드 --(테스크 구동)--> 익스큐터
  6. 익스큐터 --(테스크 실행)--> 셔플 앱 테스크 또는 결과 테스크

#### DAG 구성
태스크 종류
  * 셔플 맵 태스크(shuffle map task) : 맵리튜스의 셔플의 맵과 비슷. 각 셔플 맵 태스크는 파티셔닝 함수에 기반하여 RDD 파티션당 하나의 계산을 실행하고, 그 결과를 새로운 파티션 집합에 저장. 새로운 파티션 데이터는 셔플 맵 태스크나 결과 태스크로 구성된 다음에 스테이지에서 사용됨. 마지막 스테이지를 제외한 모든 스테이지에서 실행될 수 있음
  * 결과 태스크(result task) : 결과 태스크는 count() 액션의 결과처럼 그 결과를 사용자 프로그램에 돌려주는 마지막 스테이지에서 실행됨. 각 결과 태스크는 RDD 파티션에서 계산을 수행하고 그 결과를 드라이버에 돌려줌. 드라이버는 각 파티션의 결과를 하나로 모아서 최종 결과를 만듬(saveAsTextFile()과 같은 액션은 각각의 결과를 모으지 않고 독립적으로 저장)

#### 태스크 스케줄링
* 태스크 스케줄러에 태스크의 집합을 전송할 때 태스크 스케줄러는 해당 어플리케이션이 되는 익스큐터의 모곡을 찾고 배치 우선권이 있는 익스큐터에 각 태스크를 매핑하고 할당한다. 이어서 모든 태스크가 완료될 때까지 태스크실행이 완료된 익스큐터에 계속 태스크를 할당한다
* 태스크가 할당되면 스케줄러 백엔드는 익스큐터 백엔드에 태스크를 구동하라는 원격 메시지(akka)를 보내고, 익스큐터는 해당 태스크를 실행한다.

#### 태스크 수행(익스큐터)
1. 태스크의 JAR 및 파일 의존성을 검사하여 최신인지 확인(로컬 캐시에 존재하는지 체크)
2. 태스크 구동 메시지의 일부로 전송된 직렬화 바이트를 사용자 함수가 포함된 태스크로 역직렬화
3. 태스크 코드를 수행
> 태스크는 익스큐터와 동일한 JVM에서 실행되므로 태스크 실행에 따른 프로세스 오버헤드는 없음

---------

## 익스큐터와 클러스터 매니저
#### 클러스터 매니저
로컬(local)
  * 로컬 모드에는 드라이버와 동일한 JVM에서 실행되는 단일 익스큐터가 있음
  * 테스트, 작은 잡을 실행할 경우 사용
  * local[n](n은 스레드/코어, * 사용가능)

독립(standalone)
  * 단일 스파크 마스터와 하나 이상의 워커로 실행. 스파크 애플리케이션이 시작되면 마스터는 애플리케이션을 대신해서 모든 워커에 익스큐터 프로세스를 생성하도록 요청
  * 정적 자원 할당 정책을 사용하므로 나중에 다른 애플리케이션의 요구사항이 발생했을 때 대처하기 어려움
  * master URL: spark://host:port

메소스(mesos)
  * 미세 단위(fine-grained, Default) 모드에서 각 스파크 태스크는 메소스 태스크로 실행됨. 클러스터 자원을 매우 효율적으로 관리하지만 프로세스 구동 시 오버헤드가 있음
  * 큰 단위(Coarse-grained) 모드에서 익스큐터는 프로세스의 내부에서 해당 태스크를 실행. 즉, 스파크 애플리케이션이 실행되는 동안 익스큐터 프로세스가 클러스터의 자원을 계속 유지
  * master URL : mesos://host:port

얀(YARN)
  * 각 스파크 애플리케이션은 YARN 애플리케이션의 인스턴스며, 각 익스큐터는 자체 YARN 컨테이너에서 실행
  * 보안, 자원, 스케줄 관리에 있어 독립모드(단독)보단 우수
  * master URL : yarn-client, yarn-cluster
  * 익스큐터는 데이터 지역성 정보가 없는 상태에서 구동됨. 따라서 스파크 잡이 접근할 파일을 관리하는 데이터노드에 함께 배치되지 않는다. 따라서 데이터 지역성을 높이기 위해 배치 정책에 대한 정보를 미리 알려주는 방식을 추가해야함 (InputFormatInfo 헬퍼 클래스 이용)

#### YARN 클라이언트 모드
처리 순서
1. 드라이버 프로그램이 새로운 SparkContext 인스턴스를생성할 떄 YARN과 연결
2. SparkContext는 YARN 애플리케이션을 YARN 리소스 매니저에 제출
3. 클러스터의 노드 매니저에 YARN 컨테이너를 시작하고 스파크 ExecutorLauncher 애플리케이션 마스터를 실행
4. ExecutorLauncher 잡은 YARN 컨테이너에서 익스큐터를 시작하고, 필요한 자원을 리소스 매니저에 요청
5. 그 다음에 할당 받은 컨테이너에서 ExecutorBackend 프로세스를 구동
6. 각 익스큐터가 시작되면 반대로 SparkContext와 연결하고 자신을 등록. SparkContext는 태스크 배치 결정에 이를 활용


실행
~~~bash
# master URL은 HADOOP_CONF_DIR에 지정된 설정을 읽어옴
$ spark-shell --master yarn-client \
  --num-executors 4 \
  --executor-cores 1 \
  --executor-memory 2g
~~~

#### YARN 클러스터 모드
사용자의 드라이버 프로그램은 YARN 애플리케이션 마스터 프로세스의 내부에서 실행

~~~bash
$ spark-submit --master yarn-cluster ..
~~~
