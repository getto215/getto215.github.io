# kafka offset lag 수집
burrow를 이용해서 lag를 수집해보기
서버 : logis-ctl-01

## burrow
* https://github.com/linkedin/Burrow


-- 다운로드
https://github.com/linkedin/Burrow/releases/download/v1.1.0/Burrow_1.1.0_linux_amd64.tar.gz

-- 실행
$ ./burrow --config-dir=/data/cluster/logis/burrow/burrow/config

-- config(burrow.toml)
~~~yaml
[general]
pidfile="burrow.pid"
stdout-logfile="logs/burrow.out"
#access-control-allow-origin="mysite.example.com"

[logging]
filename="logs/burrow.log"
level="info"
maxsize=100
maxbackups=30
maxage=10
use-localtime=true
use-compression=true

[zookeeper]
servers=[ "logis-dp-01:2181", "logis-dp-02:2181", "logis-dp-03:2181" ]
timeout=6
root-path="/burrow"

[client-profile.logis-dev-client]
client-id="burrow-logis"
kafka-version="0.10.2"

[cluster.logis-dev]
class-name="kafka"
servers=[ "logis-dp-01:9092", "logis-dp-02:9092", "logis-dp-03:9092" ]
client-profile="logis-dev-client"
topic-refresh=120
offset-refresh=30

[consumer.logis-dev]
class-name="kafka"
cluster="logis-dev"
servers=[ "logis-dp-01:9092", "logis-dp-02:9092", "logis-dp-03:9092" ]
client-profile="logis-dev-client"
group-blacklist="^(console-consumer-|python-kafka-consumer-|quick-).*$"
group-whitelist=""

[consumer.logis-dev-zk]
class-name="kafka_zk"
cluster="logis-dev"
servers=[ "logis-dp-01:2181", "logis-dp-02:2181", "logis-dp-03:2181" ]
zookeeper-path="/kafka10"
zookeeper-timeout=30
group-blacklist="^(console-consumer-|python-kafka-consumer-|quick-).*$"
group-whitelist=""

[httpserver.default]
address=":8002"

[storage.default]
class-name="inmemory"
workers=20
intervals=15
expire-group=604800
min-distance=1

#[notifier.default]
#class-name="http"
#url-open="http://someservice.example.com:1467/v1/event"
#interval=60
#timeout=5
#keepalive=30
#extras={ api_key="REDACTED", app="burrow", tier="STG", fabric="mydc" }
#template-open="config/default-http-post.tmpl"
#template-close="config/default-http-delete.tmpl"
#method-close="DELETE"
#send-close=true
#threshold=1

~~~


-- 연결
http://logis-ctl-01:8002/burrow/admin

-- http endpoint
https://github.com/linkedin/Burrow/wiki/HTTP-Endpoint

* Healthcheck (GET) : /burrow/admin
* List Clusters	(GET)	: /v3/kafka
* Kafka Cluster Detail (GET) : /v3/kafka/(cluster)
* List Consumers (GET):	/v3/kafka/(cluster)/consumer
* List Cluster Topics	GET	/v3/kafka/(cluster)/topic
* Get Consumer Detail	GET	/v3/kafka/(cluster)/consumer/(group)
* Consumer Group Status	GET	/v3/kafka/(cluster)/consumer/(group)/status /v3/kafka/(cluster)/consumer/(group)/lag
* Remove Consumer Group	DELETE	/v3/kafka/(cluster)/consumer/(group)
* Get Topic Detail	GET	/v3/kafka/(cluster)/topic/(topic)
* Get General Config	GET	/v3/config
* List Cluster Modules	GET	/v3/config/cluster
* Get Cluster Module Config	GET	/v3/config/cluster/(name)
* List Consumer Modules	GET	/v3/config/consumer
* Get Consumer Module Config	GET	/v3/config/consumer/(name)
* List Notifier Modules	GET	/v3/config/notifier
* Get Notifier Module Config	GET	/v3/config/notifier/(name)
* List Evaluator Modules	GET	/v3/config/evaluator
* Get Evaluator Module Config	GET	/v3/config/evaluator/(name)
* List Storage Modules	GET	/v3/config/storage
* Get Storage Module Config	GET	/v3/config/storage/(name)
* Get Log Level	GET	/v3/admin/loglevel
* Set Log Level	POST	/v3/admin/loglevel

-- url
* cluster list : http://logis-ctl-01:8002/v3/kafka
* consumer list : http://logis-ctl-01:8002/v3/kafka/logis-dev/consumer
* consumer
  * http://logis-ctl-01:8002/v3/kafka/logis-dev/consumer/logstash-infra-logis-metrics/status


---------
## telegraf burrow plugin
* https://github.com/influxdata/telegraf/tree/master/plugins/inputs/burrow

#### config
~~~yaml
########################
## burrow (kafka offset)
########################
[[inputs.burrow]]
  servers = ["http://logis-ctl-01:8002"]
  api_prefix = "/v3/kafka"
  # response_timeout = "5s"

  ## Limit per-server concurrent connections.
  ## Useful in case of large number of topics or consumer groups.
  # concurrent_connections = 20

  clusters_include = ["logis-dev"]
  # clusters_exclude = []

  ## Filter consumer groups, default is no filtering.
  ## Values can be specified as glob patterns.
  # groups_include = []
  # groups_exclude = []

  ## Filter topics, default is no filtering.
  ## Values can be specified as glob patterns.
  # topics_include = ["infra.logis.metrics"]
  topics_exclude = ["__consumer_offsets", "__confluent.support.metrics"]
~~~
## json
~~~json
--> ## topic
{
  "timestamp": 1534905460,
  "tags": {
    "usage": "logstash",
    "topic": "infra.logis.metrics",
    "project_id": "logis",
    "partition": "0",
    "host": "logis-ctl-01",
    "cluster": "logis-dev"
  },
  "name": "burrow_topic",
  "fields": {
    "offset": 3487855
  }
}
---> ## partition
{
  "timestamp": 1534905460,
  "tags": {
    "usage": "logstash",
    "topic": "infra.logis.metrics",
    "project_id": "logis",
    "partition": "0",
    "host": "logis-ctl-01",
    "group": "logstash-test04",
    "cluster": "logis-dev"
  },
  "name": "burrow_partition",
  "fields": {
    "timestamp": 1534905459563,
    "status_code": 1,
    "status": "OK",
    "offset": 3487922,
    "lag": 0
  }
}
---> ## group
{
  "timestamp": 1534905460,
  "tags": {
    "usage": "logstash",
    "project_id": "logis",
    "host": "logis-ctl-01",
    "group": "logstash-test04",
    "cluster": "logis-dev"
  },
  "name": "burrow_group",
  "fields": {
    "total_lag": 0,
    "timestamp": 1534905459675,
    "status_code": 1,
    "status": "OK",
    "partition_count": 3,
    "offset": 3487922,
    "lag": 0
  }
}
~~~

* status code
OK = 1
NOT_FOUND = 2
WARN = 3
ERR = 4
STOP = 5
STALL = 6
unknown value will be mapped to 0

## data 확인
#### burrow_group
> select * from burrow_group  order by time desc limit 5
name: burrow_group
time                 cluster   group           host         lag offset  partition_count project_id status status_code timestamp     total_lag usage
----                 -------   -----           ----         --- ------  --------------- ---------- ------ ----------- ---------     --------- -----
2018-08-22T05:05:50Z logis-dev logstash-test04 logis-ctl-01 0   3548685 3               logis      OK     1           1534914345760 0         logstash
2018-08-22T05:05:40Z logis-dev logstash-test04 logis-ctl-01 0   3548628 3               logis      OK     1           1534914335760 0         logstash
2018-08-22T05:05:30Z logis-dev logstash-test04 logis-ctl-01 0   3548502 3               logis      OK     1           1534914315758 0         logstash


#### burrow_partition
> select * from burrow_partition where topic = 'infra.logis.metrics' order by time desc limit 5
name: burrow_partition
time                 cluster   group           host         lag offset  partition project_id status status_code timestamp     topic               usage
----                 -------   -----           ----         --- ------  --------- ---------- ------ ----------- ---------     -----               -----
2018-08-22T05:05:20Z logis-dev logstash-test04 logis-ctl-01 0   3546131 2         logis      OK     1           1534914315751 infra.logis.metrics logstash
2018-08-22T05:05:20Z logis-dev logstash-test04 logis-ctl-01 0   3548502 0         logis      OK     1           1534914315748 infra.logis.metrics logstash
2018-08-22T05:05:20Z logis-dev logstash-test04 logis-ctl-01 0   3547984 1         logis      OK     1           1534914315758 infra.logis.metrics logstash
2018-08-22T05:05:10Z logis-dev logstash-test04 logis-ctl-01 0   3548446 0         logis      OK     1           1534914305748 infra.logis.metrics logstash
2018-08-22T05:05:10Z logis-dev logstash-test04 logis-ctl-01 0   3547918 1         logis      OK     1           1534914305757 infra.logis.metrics logstash

#### burrow_topic
> select * from burrow_topic where topic = 'infra.logis.metrics' order by time desc limit 5
name: burrow_topic
time                 cluster   host         offset  partition project_id topic               usage
----                 -------   ----         ------  --------- ---------- -----               -----
2018-08-22T05:04:20Z logis-dev logis-ctl-01 3545567 2         logis      infra.logis.metrics logstash
2018-08-22T05:04:20Z logis-dev logis-ctl-01 3547983 0         logis      infra.logis.metrics logstash
2018-08-22T05:04:20Z logis-dev logis-ctl-01 3547419 1         logis      infra.logis.metrics logstash
2018-08-22T05:04:10Z logis-dev logis-ctl-01 3547983 0         logis      infra.logis.metrics logstash
2018-08-22T05:04:10Z logis-dev logis-ctl-01 3547419 1         logis      infra.logis.metrics logstash

## grafana 설정
-- offset
* SELECT difference(mean("offset")) FROM "burrow_topic" WHERE time >= now() - 30m GROUP BY time(1m), "topic" fill(previous)
* burrow_topic에 파티션별로 offset이 저장됨. sum으로 해야 정확하지만 데이터가 들어오는 타이밍이 있어서.. 늦어질 경우 그래프가 들쑥날쑥함. 따라서 mean을 사용하도록 함
* 초당 유입량은 mean을 사용했으므로, time interval과 partition 개수를 고려해서 계산해야함 (mean x #partitions / interval)

-- lag
* SELECT sum("lag") FROM "burrow_partition" WHERE time >= now() - 30m GROUP BY time(2s), "group", "topic" fill(none)
* 1개 토픽당 1개의 group_id 일 경우만 사용

-- lag가 0으로 들어오는 이슈
https://github.com/linkedin/Burrow/issues/375