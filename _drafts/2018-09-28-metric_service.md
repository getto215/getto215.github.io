# Metrics 서비스를 위한 테스트

1. influxdb 구성
2. telegraf
3. grafana 구성


## UI
* grafana: http://logis-ctl-01:3000/
* chronograf: http://logis-ctl-01:8888


## grafana
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.2.1_amd64.deb 
sudo dpkg -i grafana_5.2.1_amd64.deb

http://logis-dp-01:3000



## chronograf
* download: https://portal.influxdata.com/downloads#influxdb

wget https://dl.influxdata.com/chronograf/releases/chronograf_1.6.0_amd64.deb
sudo dpkg -i chronograf_1.6.0_amd64.deb


## Kafka 전송
* Kafka brokers : logis-dp-01~03
* topic : infra.logis.metrics

-- create
./kafka-topics --zookeeper logis-dp-01:2181,logis-dp-02:2181,logis-dp-03:2181/kafka10 --create --topic infra.logis.metrics --partitions 3 --replication-factor 2

./kafka-topics --zookeeper logis-dp-01:2181,logis-dp-02:2181,logis-dp-03:2181/kafka10 --create --topic infra.logis.plain --partitions 3 --replication-factor 2

./kafka-topics --zookeeper logis-dp-01:2181,logis-dp-02:2181,logis-dp-03:2181/kafka10 --create --topic infra.logis.log --partitions 3 --replication-factor 2

-- consumer
./kafka-console-consumer --bootstrap-server logis-dp-01:9092 --topic infra.logis.metrics --from-beginning

./kafka-console-consumer --bootstrap-server logis-dp-01:9092 --topic infra.logis.log

-- consumer list
./kafka-run-class kafka.admin.ConsumerGroupCommand --bootstrap logis-dp-01:9092,logis-dp-02:9092,logis-dp-03:9092 --new-consumer --list


-- lag
./kafka-run-class kafka.admin.ConsumerGroupCommand --bootstrap logis-dp-01:9092,logis-dp-02:9092,logis-dp-03:9092 --new-consumer --group logstash-test04 --describe



## logstash가 너무 느릴경우
* https://www.passbe.com/2016/12/21/logstash-slow-start-up-times-and-exhausting-entropy/
~~~bash
$ cat /proc/sys/kernel/random/entropy_avail
47
$ sudo apt-get install haveged
...
$ cat /proc/sys/kernel/random/entropy_avail
2432
~~~


## kafka manager
* https://github.com/yahoo/kafka-manager
* ver: 1.3.3.18

#### build
sbt clean dist

#### build packages
sbt debian:packageBin

--> dependency
sudo apt-get install fakeroot
--> systemD로 구성되어 있어 사용하지 못함




## influxdb
$ influx -precision rfc3339

* DB 커맨드
https://docs.influxdata.com/influxdb/v1.6/query_language/schema_exploration/

* tags 확인
show tags keys on logis

* field 확인
show field keys on logis

---------------
# 데이터 파이프라인 테스트
## telegraf 설정
~~~
[global_tags]
  service_name = "logis"

[agent]
  interval = "10s"
  ...

[[outputs.kafka]]
  brokers = ["logis-dp-01:9092","logis-dp-02:9092","logis-dp-03:9092"]
  topic = "infra.logis.metrics"
  required_acks = 0
  data_format = "json"
~~~

## kafka data
~~~json
{
  "timestamp": 1532476300,
  "tags": {
    "service_name": "logis",
    "host": "logis-dp-01"
  },
  "name": "system",
  "fields": {
    "uptime": 1973381
  }
}
~~~

## Logstash 설정
* logstash-output-influxdb plugin 수정

~~~yaml
# logstash.conf
output {
  influxdb {
    host => "logis-dp-01"
    port => "8086"
    db => "logis"
    telegraf_format => true # ==> add
}
~~~


## data format
telegraf에서 data_format 설정에 따라..

* influx
system,host=logis-dp-03,service_name=logis uptime_format="34 days, 23:32" 1533520170000000000

* json
{
  "fields":{
    "uptime_format":"34 days, 20:17"
  },
  "name":"system",
  "tags":{
    "host":"logis-dp-02",
    "service_name":"logis"
  },
  "timestamp":1533520550
}

* Insert data to influxdb
INSERT system,host=serverA,service_name=logis uptime=3018221i

----
## telegraf etl 테스트
data_format=json

* input
~~~
{
  "fields":{
    "load1":0.04,"load15":0.1,"load5":0.13,"n_cpus":8,"n_users":5
  },
  "name":"system",
  "tags":{
    "host":"logis-dp-01","service_name":"logis"
  },
  "timestamp":1533538240
}
~~~

* output
~~~
#(1) influx
kafka_consumer,host=logis-ctl-01,service_name=logis fields_load15=0.05,fields_load5=0.02,timestamp=1533537800,fields_n_cpus=8,fields_n_users=0,fields_load1=0 1533537801808616547

#(2) json
{
  "fields":{
    "fields_load1":0.1,"fields_load15":0.08,"fields_load5":0.12,
    "fields_n_cpus":8,"fields_n_users":5,"timestamp":1533537790
  },
  "name":"kafka_consumer",
  "tags":{
    "host":"logis-ctl-01",
    "service_name":"logis"
  },
  "timestamp":1533537790
}
~~~

## 참고
* https://github.com/influxdata/influxdb-ruby
* https://github.com/logstash-plugins/logstash-output-influxdb
* https://www.elastic.co/guide/en/logstash/5.4/plugins-outputs-influxdb.html
* ruby 가이드
  * https://www.ruby-lang.org/ko/documentation/quickstart/3/
  * https://docs.google.com/document/d/15yEpi2ZMB2Lld5lA1TANt13SJ_cKygP314cqyKhELwQ/preview


---------------
## 적재 방식
#### influxdb 적재 방법
1. 프로젝트별로 influxDB에 database를 만든다.
  * 프로젝트 수만큼 db가 생성된다. 거의 200개.. 미리 알아서 이만큼을 넣어놔야할까?
2. 기본 데이터(zabbix)는 하나의 influxDB에 넣는다. 그 외 프로젝트별 매트릭은 요청 시 별도의 db를 구성한다.
  * SDI의 서버별 모니터링에 유용할 듯 보임. 기본 데이터에 ip가 들어있으므로.
  * 적재되는 데이터는 정규식을 이용해서 쉽게 가져올 수 있도록 구성해야함.
  * 다른 데이터베이스의 데이터와 join할 일이 있나?
  * 가공한 데이터는 어디에 넣는게 맞을까
  * retention 주기도 동일해져야함..
  * 별도 db 정보는 telegraf 세팅 시 project_id(group_id)로 넣어야하고, 인증된 데이터만 들어갈 수 있도록 etl에서 filtering한다. 마지막으로 사용자는 해당 db에 접근할 수 있는 계정을 부여받는다. (local)
  * 사용자는 최소 1개(zabbix)와 추가 db를 사용해야한다.
  * filtering에 필요한 데이터는 별도의 db로 처리하지 않고, 그냥 array로 넣어서 처리함. db로 연동. 짧은 주기로 긁어오지 않기 위해 별도의 파일로 다운받아서 처리하도록 함.

---------------
## Logstash filter
* 허용된 group_id만 데이터를 적재할 수 있어야함
* 데이터는 mysql에 저장(meta)
* input과 filter, output이 어떻게 돌아가는지 확인할 것. 하나의 row별로 전체 프로세스가 돌아가는건지.. 작업단위별로 처리가 되는건지 확인이 필요.

* 참고
  * https://www.elastic.co/guide/en/logstash/5.4/lookup-enrichment.html
  * https://www.elastic.co/guide/en/logstash/current/plugins-inputs-exec.html
  * 

translate와 exec를 사용함.
1. input
  * exec를 이용하여 mysql에 데이터를 가져와서 json으로 저장함
2. filter
  * translate를 이용하여 데이터를 join함. <-- 이것부터 해보기

#### 필수 tags 추출
* tags의 데이터를 추출해서.. service_name과 host 정보를 @metadata로 처리해야함.
  * --> 데이터 정합성(?), 어디에 저장할지 정하기 위해..
* service_name : 이거말고.. 적당한거 찾아야함.
  * 이 이름으로 influxDB의 db명이 지정되어야함
  * project_id <-- 이걸로. DB명이 됨
* host : 필수 처리는 X


-----
#### translate
* https://www.elastic.co/guide/en/logstash/5.4/plugins-filters-translate.html

* 설치
~~~
$ ./logstash-plugin install logstash-filter-translate
Validating logstash-filter-translate
Installing logstash-filter-translate
Installation successful

$ ./logstash-plugin list --verbose | grep translate
logstash-filter-translate (3.1.0)
~~~

* 설정
~~~
filter {
  if [tags][project_id] {
    translate {
      dictionary_path => "${LS_HOME}/config/logis_project_ids.json"
      field => "[tags][project_id]"
      destination => "[@metadata][influxdb]"
      add_field => { "[@metadata][project_id]" => "%{[tags][project_id]}" }
    }
  }
}
output {
#  stdout { codec => rubydebug }
   if [@metadata][influxdb] {
     file { path => "/tmp/logstash.%{[@metadata][influxdb][0]}" }
     file { path => "/tmp/logstash.%{[@metadata][influxdb][1]}" }
   } else {
     file { path => "/tmp/logstash.unknown" }
   }
#  influxdb {
#    host => "logis-dp-01"
#    port => "8086"
#    db => "logis"
#    telegraf_format => true
#  }
}
~~~

dynamic db 세팅을 위해 소스 수정 필요
https://github.com/logstash-plugins/logstash-output-influxdb/issues/71

## logstash offline plugin install
* https://www.elastic.co/guide/en/logstash/5.5/offline-plugins.html
~~~
./bin/logstash-plugin prepare-offline-pack --output logstash-output-influxdb --overwrite
./bin/logstash-plugin prepare-offline-pack logstash-filter-translate logstash-output-influxdb
~~~


---------------
## logstash 세팅

-- plugin 설치. influxdb
./logstash-plugin install logstash-output-influxdb

./logstash-plugin list --verbose | grep influxdb

-- plugin 소스 덮어씌우기
/data/cluster/logis/logstash-5.6.11/vendor/bundle/jruby/1.9/gems/logstash-output-influxdb

-- 패키지를 offline으로 만들기
./bin/logstash-plugin prepare-offline-pack logstash-output-influxdb

---------------
## file count
Logstash에 page 파일이 얼마나 있는지 체크
* https://github.com/influxdata/telegraf/tree/master/plugins/inputs/filecount
--> 1.8 이후에나 나올 듯. exec를 이용

#### filecount.sh
~~~bash
#!/bin/bash

FILES="/data/cluster/logis/logstash5/data/queue/main/page.*"
COUNT=`/bin/ls ${FILES} | /usr/bin/wc -l`

echo "{\"path\": \"${FILES}\", \"file_count\":${COUNT}}"
~~~

#### telegraf.conf
~~~yaml
#----------------
# logstash data file count
#----------------
[[inputs.exec]]
  commands = ["sh /etc/telegraf/filecount.sh"]
  timeout = "5s"
  interval = "10s"
  name_override = "logstash_page"
  tag_keys = ["path"]
  data_format = "json"
~~~

---------------
## net plugin
network in/out 체크용

~~~yaml
[[inputs.net]]
  interfaces = ["eth*"]
~~~

---------------
## 로깅 처리. td-agent
#### 테스트 서버 구성
* elasticsearch : logis-dp-02
* kibana : logis-dp-02
* logstash : logis-dp-03

#### elasticsearch 설치
* logis-dp-02
* package install guide : https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html

* pkg 설치할 때 subprocess 오류가 나는 경우
java path가 잡히지 않아서 발생. root의 .bashrc에 java를 잡아준 뒤 root 계정으로 설치함

* 테스트
~~~bash
ubuntu@logis-dp-02:~/usr/pkgs$ curl -XGET "http://localhost:9200/"
{
  "name" : "R7HgDOH",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "kWug72vEQyKGsnNUUJ6fgA",
  "version" : {
    "number" : "6.4.0",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "595516e",
    "build_date" : "2018-08-17T23:18:47.308994Z",
    "build_snapshot" : false,
    "lucene_version" : "7.4.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
~~~

#### kibana 설치
* logis-dp-02
* package install guide : https://www.elastic.co/guide/en/kibana/current/deb.html

* bind 설정
vi /etc/kibana/kibana.yml
~~~yaml
server.host: "0.0.0.0"
~~~


#### td-agent 설치
* https://docs.fluentd.org/v1.0/articles/install-by-deb
* download : wget http://packages.treasuredata.com.s3.amazonaws.com/3/ubuntu/trusty/pool/contrib/t/td-agent/td-agent_3.2.0-0_amd64.deb

* parsing : http://fluentular.herokuapp.com/
* ruby regex : http://rubular.com/
* multiline : https://docs.fluentd.org/v1.0/articles/parser_multiline
* output config : https://docs.fluentd.org/v1.0/articles/output-plugin-overview
* kafka plugin : https://github.com/fluent/fluent-plugin-kafka

* Install
~~~bash
# install
$ sudo dpkg -i td-agent_3.2.0-0_amd64.deb

# plugin list
$ td-agent-gem list
~~~


#### telegraf logging
* path: /var/log/telegraf/telegraf.log
* log
~~~bash
2018-09-09T23:01:32Z I! Loaded processors:
2018-09-09T23:01:32Z I! Loaded outputs: kafka
2018-09-09T23:01:32Z D! Loaded outputs: kafka
2018-09-09T23:01:32Z E! Loaded outputs: kafka
...
~~~
time, loglevel, msg

~~~yaml
<source>
  @type tail
  path /var/log/telegraf/telegraf.log
  pos_file /var/log/td-agent/buffer/telegraf.pos
  tag log.telegraf
  refresh_interval 10s            # default: 60s
  multiline_flush_interval 5s     # default: 5s
  <parse>
    @type multiline
    format_firstline /\d{4}-\d{1,2}-\d{1,2}/
    format1 /^(?<time>[^ ]*) (?<loglevel>[^\!]*)! (?<msg>.*)/
    keep_time_key true
    time_type string
    time_format %Y-%m-%dT%H:%M:%SZ
  </parse>
</source>

<filter log.**>
  @type record_transformer
  <record>
    service_name ${tag_parts[1]}   # [Required]
    logtime ${Time.now.iso8601}    # [Required]
    server "#{Socket.gethostname}"
  </record>
  enable_ruby
</filter>

##<match tmp.**>
##  @type file
##  path /tmp/td-agent.data
##  <format>
##    @type json
##  </format>
##</match>

<match log.**>
  @type kafka_buffered
  brokers logis-dp-01:9092,logis-dp-02:9092,logis-dp-03:9092
  default_topic infra.logis.log
  output_data_type json
  max_send_retries 3
  required_acks 0
  <buffer>
  #  @type file
  #  path /var/log/td-agent/buffer/kafka
    flush_interval 10s
  </buffer>
</match>
~~~


#### kafka logging
* path : /data/cluster/logis/kafka/logs/server.log
* log
~~~
[2018-08-20 14:48:27,358] INFO Deleting index /data/cluster/logis/kafka/data/__consumer_offsets-27/00000000000000000000.timeindex.deleted (kafka.log.TimeIndex)
[2018-08-20 14:56:36,968] INFO [Group Metadata Manager on Broker 1]: Removed 0 expired offsets in 0 milliseconds. (kafka.coordinator.GroupMetadataManager)
~~~
* parsing
~~~
^\[(?<time>[^\]]*)\] (?<loglevel>[^ ]*) (?<msg>.*)
~~~

* config
~~~yaml
<source>
  @type tail
  path /data/cluster/logis/kafka/logs/server.log
  pos_file /var/log/td-agent/buffer/kafka.pos
  tag log.kafka
  refresh_interval 10s            # default: 60s
  multiline_flush_interval 5s     # default: 5s
  <parse>
    @type multiline
    format_firstline /\[\d{4}-\d{1,2}-\d{1,2}/
    format1 /^\[(?<time>[^\]]*)\] (?<loglevel>[^ ]*) (?<msg>.*)/
    keep_time_key true
    time_type string
    time_format %Y-%m-%d %H:%M:%S,%L
  </parse>
</source>

<filter log.**>
  @type record_transformer
  <record>
    service_name ${tag_parts[1]}
    logtime ${Time.now.iso8601}
    server "#{Socket.gethostname}"
    cluster "dev"
  </record>
  enable_ruby
</filter>

<match log.**>
  @type kafka_buffered
  brokers logis-dp-01:9092,logis-dp-02:9092,logis-dp-03:9092
  default_topic infra.logis.log
  output_data_type json
  max_send_retries 3
  required_acks 0
  <buffer>
    flush_interval 10s
  </buffer>
</match>
~~~

#### logstash5 logging
* 체크사항
  * 파일이 없을 경우.. 에러 최소화
  * sudo 권한 필요
* path : /var/log/upstart/
* log
~~~
[2018-09-10T13:54:32,718][INFO ][org.apache.kafka.clients.consumer.internals.AbstractCoordinator] (Re-)joining group logstash-infra-logis-log
~~~
([A-Z])\w+

* parsing
~~~
^\[(?<time>[^\]]*)\]\[(?<loglevel>[^\]]*)\]\[(?<proc>[^\]]*)\] (?<msg>.*)
~~~

* conf
~~~yaml
<source>
  @type tail
  path /var/log/upstart/logstash5.log
  pos_file /var/log/td-agent/buffer/logstash5.pos
  tag log.logstash5
  refresh_interval 10s            # default: 60s
  multiline_flush_interval 5s     # default: 5s
  <parse>
    @type multiline
    format_firstline /\[\d{4}-\d{1,2}-\d{1,2}/
    format1 /^\[(?<time>[^\]]*)\]\[(?<loglevel>[^\]]*)\]\[(?<proc>[^\]]*)\] (?<msg>.*)/
    keep_time_key true
    time_type string
    time_format %Y-%m-%dT%H:%M:%S,%L
  </parse>
</source>

<filter log.**>
  @type record_transformer
  <record>
    service_name ${tag_parts[1]}   # [Required]
    logtime ${Time.now.iso8601}    # [Required]
    server "#{Socket.gethostname}"
    cluster "dev"
  </record>
  enable_ruby
</filter>

<match log.**>
  @type kafka_buffered
  brokers logis-dp-01:9092,logis-dp-02:9092,logis-dp-03:9092
  default_topic infra.logis.log
  output_data_type json
  max_send_retries 3
  required_acks 0
  <buffer>
  #  @type file
  #  path /var/log/td-agent/buffer/kafka
    flush_interval 10s
  </buffer>
</match>
~~~

---------------
## kafka producer count
https://docs.confluent.io/current/kafka/monitoring.html#producer-metrics

kafka broker에 연결된 producer 체크가 가능한지 테스트
서버 : logis-dp-01
jmxterm 이용

kafka.producer:type=producer-metrics,client-id=([-.w]+)

JMX_PORT=19998 ./kafka-console-producer --broker-list logis-dp-01:9092,logis-dp-02:9092,logis-dp-03:9092 --topic infra.logis.plain

--> producer를 켜야.. 가능함. 따라서 사용할 수 없음

---------------
## Process monitoring
* https://github.com/influxdata/telegraf/tree/master/plugins/inputs/net_response

~~~yaml
[[inputs.net_response]]
  protocol = "tcp"
  address = "localhost:80"
  # timeout = "1s"
~~~

#### metrics
* tags:
  * server
  * port
  * protocol
  * result
* fields:
  * response_time (float, seconds)
  * success (int) # success 0, failure 1
  * result_code (int, success = 0, timeout = 1, connection_failed = 2, read_failed = 3, string_mismatch = 4)



---------------
## todo
* kafka 대시보드 구성 (datadog)
  * https://www.datadoghq.com/blog/monitoring-kafka-performance-metrics/


* kafka-consumer를 바로 telegraf로 쏠 수 있을까?


---------------

process 모니터링이 필요..
