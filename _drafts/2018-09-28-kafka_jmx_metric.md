# Telegraf를 이용한 kafka jmx metric 수집
수집 항목
- JMX
- offset

---------
## JMX
jolokia로 할까.. jolokia proxy로 할까..
설치 편의를 봐선.. proxy가 나을 것 같은데.. 세팅해서 사용해보기.

* plugin: https://github.com/influxdata/telegraf/tree/master/plugins/inputs/jolokia2

#### jolokia 설치
https://jdm.kr/blog/231

-- tomcat download
http://apache.mirror.cdnetworks.com/tomcat/tomcat-8/v8.5.32/bin/apache-tomcat-8.5.32.tar.gz

-- jolokia 1.3.4
http://repo1.maven.org/maven2/org/jolokia/jolokia-war/1.3.4/jolokia-war-1.3.4.war

-- 세팅
# tomcat의 webapp에 넣기
$ mv jolokia-war-1.3.4.war ./webapps/

# 이름 변경
mv jolokia-war-1.3.4.war jolokia.war

-- 포트 변경
conf/server.xml
8080 --> 8001

-- 시작
./bin/catalina.sh start

-- 확인
http://logis-dev-01:8001/jolokia/version

-- 실행 파일
export JAVA_HOME=/usr/lib/jvm/java-7-sun
export PATH=$JAVA_HOME/bin:$PATH



####  jmx list
* 토픽별
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec OneMinuteRate
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec OneMinuteRate
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec OneMinuteRate
kafka.server:type=BrokerTopicMetrics,name=BytesRejectedPerSec OneMinuteRate
kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec OneMinuteRate
kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec OneMinuteRate

* 네트워크
kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchConsumer OneMinuteRate
kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchFollower OneMinuteRate
kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Produce OneMinuteRate
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer OneMinuteRate
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchFollower OneMinuteRate
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce OneMinuteRate

* 로그 flush
kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs OneMinuteRate


#### telegraf 설정
~~~yml
########################
## Jolokia proxy
########################
## Read JMX metrics from a Jolokia REST proxy endpoint
[[inputs.jolokia2_proxy]]
  url = "http://logis-ctl-01:8001/jolokia"
  # response_timeout = "5s"

  [[inputs.jolokia2_proxy.target]]
    url = "service:jmx:rmi:///jndi/rmi://logis-dp-01:19999/jmxrmi"

  [[inputs.jolokia2_proxy.target]]
    url = "service:jmx:rmi:///jndi/rmi://logis-dp-02:19999/jmxrmi"
  
  [[inputs.jolokia2_proxy.target]]
    url = "service:jmx:rmi:///jndi/rmi://logis-dp-03:19999/jmxrmi"

  ##----------------------
  ## BrokerTopicMetrics
  ##----------------------
  #kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_bytes_in_per_sec"
    mbean = "kafka.server:name=BytesInPerSec,topic=*,type=BrokerTopicMetrics"
    tag_keys = ["topic"]

  #kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_bytes_out_per_sec"
    mbean = "kafka.server:name=BytesOutPerSec,topic=*,type=BrokerTopicMetrics"
    tag_keys = ["topic"]

  #kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_messages_in_per_sec"
    mbean = "kafka.server:name=MessagesInPerSec,topic=*,type=BrokerTopicMetrics"
    tag_keys = ["topic"]

  #kafka.server:type=BrokerTopicMetrics,name=BytesRejectedPerSec OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_bytes_rejected_per_sec"
    mbean = "kafka.server:name=BytesRejectedPerSec,topic=*,type=BrokerTopicMetrics"
    tag_keys = ["topic"]

  #kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_failed_fetch_requests_per_sec"
    mbean = "kafka.server:name=FailedFetchRequestsPerSec,topic=*,type=BrokerTopicMetrics"
    tag_keys = ["topic"]

  #kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_failed_produce_requests_per_sec"
    mbean = "kafka.server:name=FailedProduceRequestsPerSec,topic=*,type=BrokerTopicMetrics"
    tag_keys = ["topic"]

  ##----------------------
  ## RequestMetrics
  ##----------------------
  #kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchConsumer OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_requests_per_sec"
    mbean = "kafka.network:name=RequestsPerSec,request=FetchConsumer,type=RequestMetrics"
    tag_keys = ["request"]
  
  #kafka.network:type=RequestMetrics,name=RequestsPerSec,request=FetchFollower OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_requests_per_sec"
    mbean = "kafka.network:name=RequestsPerSec,request=FetchFollower,type=RequestMetrics"
    tag_keys = ["request"]

  #kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Produce OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_requests_per_sec"
    mbean = "kafka.network:name=RequestsPerSec,request=Produce,type=RequestMetrics"
    tag_keys = ["request"]

  #kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_total_time_ms"
    mbean = "kafka.network:name=TotalTimeMs,request=FetchConsumer,type=RequestMetrics"
    tag_keys = ["request"]

  #kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchFollower OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_total_time_ms"
    mbean = "kafka.network:name=TotalTimeMs,request=FetchFollower,type=RequestMetrics"
    tag_keys = ["request"]

  #kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_total_time_ms"
    mbean = "kafka.network:name=TotalTimeMs,request=Produce,type=RequestMetrics"
    tag_keys = ["request"]

  ##----------------------
  ## LogFlushStats
  ##----------------------
  #kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs OneMinuteRate
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_log_flush_rate_and_time_ms"
    mbean = "kafka.log:name=LogFlushRateAndTimeMs,type=LogFlushStats"

  ##----------------------
  ## Management
  ##----------------------
  # kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
  # * Number of under-replicated partitions (| ISR | < | all replicas |). Alert if value is greater than 0.
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_under_replicated_partitions"
    mbean = "kafka.server:name=UnderReplicatedPartitions,type=ReplicaManager"

  # kafka.controller:type=KafkaController,name=OfflinePartitionsCount
  # * Number of partitions that don’t have an active leader and are hence not writable or readable. Alert if value is greater than 0.
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_offline_partitions_count"
    mbean = "kafka.controller:name=OfflinePartitionsCount,type=KafkaController"

  # kafka.controller:type=KafkaController,name=ActiveControllerCount
  # * Number of active controllers in the cluster. Alert if the aggregated sum across all brokers in the cluster is anything other than 1 because there should be exactly one controller per cluster.
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_active_controller_count"
    mbean = "kafka.controller:name=ActiveControllerCount,type=KafkaController"

  # kafka.server:type=ReplicaManager,name=LeaderCount
  # * Number of leaders on this broker. This should be mostly even across all brokers. If not, set auto.leader.rebalance.enable to true on all brokers in the cluster.
  [[inputs.jolokia2_proxy.metric]]
    name  = "kafka_leader_count"
    mbean = "kafka.server:name=LeaderCount,type=ReplicaManager"

~~~

--------

FiveMinuteRate":  0.003050878228299864,
"Mean":           6.106994692307692,
"MeanRate":       0.00002359200273293151,
"OneMinuteRate":  0.000009535048314160196

--------
## float test
influxdb에 매우 긴 float이 입력되면.. 잘안되는 것 같음
INSERT tmp_f,cipher=10 f=1.0000000001
INSERT tmp_f,cipher=15 f=1.000000000000001 <-- 여기까지만 제대로
INSERT tmp_f,cipher=20 f=1.00000000000000000001
INSERT tmp_f,cipher=16 f=1.0000000000000001 <-- 1
INSERT tmp_f,cipher=16 f=1.00000 00000 00000 8 <-- 1.00000 00000 00000 9
INSERT tmp_f,cipher=16 f=1.00000 00000 00000 7 <-- 1.00000 00000 00000 7
INSERT tmp_f,cipher=16 f=1.00000 00000 00000 6 <-- 1.00000 00000 00000 7  <== ?
==> 최대로 나오는건 소수점 이하 16자리까지. 올림되는게.. 이상함..

INSERT tmp_f one=2.2286337782676267e-7
INSERT tmp_f one=4.2093455e-08

----------
## kafka jmx 모니터링 항목
https://docs.confluent.io/current/kafka/monitoring.html#broker-metrics

### Broker Metrics
#### 필수 >>
* kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
  * Number of under-replicated partitions (| ISR | < | all replicas |). Alert if value is greater than 0.
  * 추출 값 : Value
  * 0보다 크면 문제..
* kafka.controller:type=KafkaController,name=OfflinePartitionsCount
  * Number of partitions that don’t have an active leader and are hence not writable or readable. Alert if value is greater than 0.
* kafka.controller:type=KafkaController,name=ActiveControllerCount
  * Number of active controllers in the cluster. Alert if the aggregated sum across all brokers in the cluster is anything other than 1 because there should be exactly one controller per cluster.

#### 선택 (이미 추가된 bytes-in 등은 제외)
* kafka.server:type=ReplicaManager,name=LeaderCount
  * Number of leaders on this broker. This should be mostly even across all brokers. If not, set auto.leader.rebalance.enable to true on all brokers in the cluster.


----------
## jmxtrans 테스트
필요한 jmx metric을 확인하기 위해 간단히 체크할 수 있는 환경을 구성함

* http://wiki.cyclopsgroup.org/jmxterm/
* 구성 위치 : logis-dp-01

-- 다운로드
https://github.com/jiaqi/jmxterm/releases

-- 실행
$ java -jar jmxterm-1.0.0-uber.jar
$ open localhost:19999

# bean 리스트
$> beans

# 추출
$> bean kafka.server:name=BytesInPerSec,topic=analysis.zabbix.log,type=BrokerTopicMetrics
$> info
$> get OneMinuteRate
...

# 한 줄로..
$> get -s -b kafka.server:name=BytesInPerSec,topic=analysis.zabbix.log,type=BrokerTopicMetrics OneMinuteRate

