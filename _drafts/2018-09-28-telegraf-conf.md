# Telegraf Configuration

[global_tags]
  project_id = "logis"
  cluster = "dev"
  usage = "logstash"

# Configuration for telegraf agent
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1200
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  debug = false
  quiet = false
  logfile = ""
  hostname = ""
  omit_hostname = false


###############################################################################
#                            OUTPUT PLUGINS                                   #
###############################################################################
[[outputs.kafka]]
  brokers = ["logis-dp-01:9092","logis-dp-02:9092","logis-dp-03:9092"]
  topic = "infra.logis.metrics"
  required_acks = 0
  data_format = "json"

###############################################################################
#                            INPUT PLUGINS                                    #
###############################################################################

# Read metrics about cpu usage
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false

# Read metrics about disk usage by mount point
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs"]

[[inputs.diskio]]


# Get kernel statistics from /proc/stat
[[inputs.kernel]]
  # no configuration

# Read metrics about memory usage
[[inputs.mem]]
  # no configuration

# Get the number of processes and group them by status
[[inputs.processes]]
  # no configuration

# Read metrics about swap memory usage
[[inputs.swap]]
  # no configuration

# Read metrics about system load & uptime
[[inputs.system]]
  # no configuration

# Collect bond interface status, slaves statuses and failures count
[[inputs.bond]]


# # Get kernel statistics from /proc/vmstat
[[inputs.kernel_vmstat]]

# Read TCP metrics such as established, time wait and sockets counts.
[[inputs.netstat]]


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

########################
## Zookeeper
########################
[[inputs.zookeeper]]
#   ## An array of address to gather stats about. Specify an ip or hostname
#   ## with port. ie localhost:2181, 10.0.0.1:2181, etc.
#
#   ## If no servers are specified, then localhost is used as the host.
#   ## If no port is specified, 2181 is used
   servers = ["logis-dp-01:2181","logis-dp-02:2181","logis-dp-03:2181"]
#
#   ## Timeout for metric collections from all servers.  Minimum timeout is "1s".
   timeout = "10s"
#
#   ## Optional TLS Config
#   # enable_tls = true
#   # tls_ca = "/etc/telegraf/ca.pem"
#   # tls_cert = "/etc/telegraf/cert.pem"
#   # tls_key = "/etc/telegraf/key.pem"
#   ## If false, skip chain & host verification
#   # insecure_skip_verify = true

########################
## elasticsearch
########################
[[inputs.elasticsearch]]
#   ## specify a list of one or more Elasticsearch servers
#   # you can add username and password to your url to use basic authentication:
#   # servers = ["http://user:pass@localhost:9200"]
  servers = ["http://admin:admin@plat-dev-05:9200"]
#
#   ## Timeout for HTTP requests to the elastic search server(s)
#   http_timeout = "5s"
#
#   ## When local is true (the default), the node will read only its own stats.
#   ## Set local to false when you want to read the node stats from all nodes
#   ## of the cluster.
#   local = true
#
#   ## Set cluster_health to true when you want to also obtain cluster health stats
#   cluster_health = false
#
#   ## Adjust cluster_health_level when you want to also obtain detailed health stats
#   ## The options are
#   ##  - indices (default)
#   ##  - cluster
#   # cluster_health_level = "indices"
#
#   ## Set cluster_stats to true when you want to also obtain cluster stats from the
#   ## Master node.
#   cluster_stats = false
#
#   ## node_stats is a list of sub-stats that you want to have gathered. Valid options
#   ## are "indices", "os", "process", "jvm", "thread_pool", "fs", "transport", "http",
#   ## "breaker". Per default, all stats are gathered.
#   # node_stats = ["jvm", "http"]
#
#   ## Optional TLS Config
#   # tls_ca = "/etc/telegraf/ca.pem"
#   # tls_cert = "/etc/telegraf/cert.pem"
#   # tls_key = "/etc/telegraf/key.pem"
#   ## Use TLS but skip chain & host verification
#   # insecure_skip_verify = false

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
