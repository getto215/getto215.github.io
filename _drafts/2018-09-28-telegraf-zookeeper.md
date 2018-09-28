# telegraf를 이용하여 zookeeper 매트릭을 수집

* https://github.com/influxdata/telegraf/tree/master/plugins/inputs/zookeeper

## config

## data
#### json
{"fields":{"approximate_data_size":15317,"avg_latency":0,"ephemerals_count":5,"max_file_descriptor_count":65536,"max_latency":12,"min_latency":0,"num_alive_connections":6,"open_file_descriptor_count":32,"outstanding_requests":0,"packets_received":2892967,"packets_sent":2892982,"version":"3.4.6-1569965","watch_count":269,"znode_count":163},"name":"zookeeper","tags":{"cluster":"dev","host":"logis-ctl-01","port":"2181","project_id":"logis","server":"logis-dp-01","state":"follower","usage":"logstash"},"timestamp":1534915940}
{"fields":{"approximate_data_size":15317,"avg_latency":0,"ephemerals_count":5,"max_file_descriptor_count":65536,"max_latency":16,"min_latency":0,"num_alive_connections":2,"open_file_descriptor_count":28,"outstanding_requests":0,"packets_received":8881,"packets_sent":8880,"version":"3.4.6-1569965","watch_count":0,"znode_count":163},"name":"zookeeper","tags":{"cluster":"dev","host":"logis-ctl-01","port":"2181","project_id":"logis","server":"logis-dp-03","state":"follower","usage":"logstash"},"timestamp":1534915940}

#### influxdb
> select * from zookeeper limit 100
name: zookeeper
time                 approximate_data_size avg_latency cluster ephemerals_count followers host         max_file_descriptor_count max_latency min_latency num_alive_connections open_file_descriptor_count outstanding_requests packets_received packets_sent pending_syncs port project_id server      state    synced_followers usage    version       watch_count znode_count
----                 --------------------- ----------- ------- ---------------- --------- ----         ------------------------- ----------- ----------- --------------------- -------------------------- -------------------- ---------------- ------------ ------------- ---- ---------- ------      -----    ---------------- -----    -------       ----------- -----------
2018-08-22T05:25:30Z 15317                 0           dev     5                          logis-ctl-01 65536                     12          0           6                     32                         0                    2892449          2892464                    2181 logis      logis-dp-01 follower                  logstash 3.4.6-1569965 269         163
2018-08-22T05:25:30Z 15317                 0           dev     5                          logis-ctl-01 65536                     16          0           2                     28                         0                    8635             8634                       2181 logis      logis-dp-03 follower                  logstash 3.4.6-1569965 0           163
2018-08-22T05:25:30Z 15317                 0           dev     5                2         logis-ctl-01 65536                     10          0           3                     31                         0                    1267329          1267389      0             2181 logis      logis-dp-02 leader   2                logstash 3.4.6-1569965 18          163
2018-08-22T05:25:40Z 15317                 0           dev     5                          logis-ctl-01 65536                     16          0           2                     28                         0                    8641             8640                       2181 logis      logis-dp-03 follower                  logstash 3.4.6-1569965 0           163


## metric
~~~
zookeeper
tags:
  server
  port
  state
fields:
  approximate_data_size (integer)
  avg_latency (integer)           -->
  ephemerals_count (integer)
  max_file_descriptor_count (integer)
  max_latency (integer)           -->
  min_latency (integer)           -->
  num_alive_connections (integer) -->
  open_file_descriptor_count (integer)
  outstanding_requests (integer)  -->
  packets_received (integer)      -->
  packets_sent (integer)          -->
  version (string)
  watch_count (integer)
  znode_count (integer)
  followers (integer, leader only)
  synced_followers (integer, leader only)
  pending_syncs (integer, leader only)
~~~

* datadog metric
  * https://docs.datadoghq.com/integrations/zk/#data-collected