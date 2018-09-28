# Logstash를 이용해서 telegraf 데이터를 influxdb에 적재
Telegraf로 생성되는 클라이언트의 매트릭 데이터를 influxDB에 적재하고자 함.
Telegraf 데이터는 우선 Kafka에 적재되기 때문에 Kafka 데이터를 influxDB로 적재하는 ETL이 필요함.
ETL로 Telegraf를 이용해도 되지만 기존에 로그를 Logstash로 처리하고 있기 때문에 SW 추가를 피하고자 했음.
그리고 Telegraf의 데이터 포맷을 influx가 아닌 json으로 처리하여 전체 플랫폼의 데이터 포맷을 통일하고자 했음 (데이터 활용)
github에 있는 logstash-output-influxdb는 telegraf의 데이터를 소스로 처리가 안됨 (nested field)
따라서 telegraf outputs.kafka의 data_format을 json으로 설정한 뒤 그에 발생하는 데이터를 기준으로 플러그인을 수정함

* 플러그인을 수정하지 않고 logstash filter를 이용해서.. 처리할 수도 있을 듯함.
  * 하지만 값들을 float으로 변환하는 건 좀 어려울 듯..


## Flow
Telegraf --> Kafka --> Logstash --> InfluxDB

## Telegraf data format (outputs.kafka)
#### data_format: influx
* 데이터 구성: `measurement`,`tags` `field` `timestamp`
* Int는 숫자 뒤에 i가 붙음
~~~
system,host=logis-dp-01,service_name=logis load1=0.07,load5=0.09,load15=0.06,n_cpus=8i,n_users=6i 1533787970000000000
~~~

#### data_format: json
* 데이터 구성: `name`,`field`,`tags`,`timestamp`
* json은 데이터타입을 지정할 수 없어 숫자형이 정확히 int인지 float인지 확인할 수 없음.
~~~
{
  "name":"system",
  "fields":{"load1":0.06,"load15":0.13,"load5":0.12,"n_cpus":8,"n_users":6},
  "tags":{"host":"logis-dp-01","service_name":"logis"},
  "timestamp":1533801270
}
~~~

## logstash-output-influxdb plugin 커스터마이징
* github : https://github.com/logstash-plugins/logstash-output-influxdb
* 수정
  * telegraf에서 발생하는 json 포맷에 맞춰 influxDB에 적재되도록 함.
  * json의 숫자형 데이터 타입을 확정하기 어렵고, 값이 추가될 때마다 반영하기 어려우므로 모두 float으로 처리되도록 함(임기응변;)
* 참고 : https://www.elastic.co/guide/en/logstash/current/_how_to_write_a_logstash_output_plugin.html


#### logstash.conf 옵션 추가
* telegraf_format (default: false) : Telegraf 설정을 적용할 것인지
~~~yaml
output {
  influxdb {
    host => "logis-dp-01"
    port => "8086"
    db => "logis"
    telegraf_format => true
}
~~~


#### influxdb.rb
~~~ruby
class LogStash::Outputs::InfluxDB < LogStash::Outputs::Base
  include Stud::Buffer
  config_name "influxdb"
  # ...
  # Add config(@getto)
  config :telegraf_format, :validate => :boolean, :default => false
  
  # ...
  public
  def receive(event)
    @logger.debug? and @logger.debug("Influxdb output: Received event: #{event}")

    # Add condition(@getto, 20180808)
    if @telegraf_format
      event_hash = telegraf_proc(event)
    else
      # default receive process
      time  = timestamp_at_precision(event.timestamp, @time_precision.to_sym)
      # ...

    # telegraf receive (@getto, 20180808)
  def telegraf_proc(event)
    @logger.debug? and @logger.debug("Telegraf receive is running.")
    
    #{
    #  "fields":{"uptime_format":"37 days,  1:09"},
    #  "name":"system",
    #  "tags":{"host":"logis-dp-01","service_name":"logis"},
    #  "timestamp":153 370 3860
    #}
    e = event.to_hash
    
    # -- time
    # timestamp 키가 없을 경우 logstash에서 발생하는 @timestamp를 이용하도록 함
    if e.has_key?('timestamp')
      time  = timestamp_at_precision(e['timestamp'], @time_precision.to_sym)
    else
      logger.warn("Cannot find timestmp. Using event @timestamp.")
      time  = timestamp_at_precision(event.timestamp, @time_precision.to_sym)
    end
    
    # -- field
    # field를 value(point)로 저장. int를 float으로 변환함.
    if e.has_key?('fields')
      point = Hash[ e['fields'].map do |k,v|
                [event.sprintf(k), (
                  if v.is_a?(Integer); v.to_f
                  elsif v.is_a?(String); event.sprintf(v)
                  else; v
                  end
                )]
      end]
      
    else
      logger.error("Cannot find any points..")
    end

    exclude_fields!(point)
    coerce_values!(point)
    
    # -- measurement
    if e.has_key?('name')
      measurement_name = e['name']
    else
      logger.error("Cannot find measurement. set default measurement value: #{@measurement}")
      measurement_name = @measurement
    end

    event_hash = {
      :series => event.sprintf(measurement_name),
      :timestamp => time,
      :values => point
    }

    # -- tags
    if e.has_key?('tags')
      event_hash[:tags] = e['tags'] unless e['tags'].empty?
    end

    event_hash
  end # def telegraf_proc

  #...

  # dynamic database
  def dowrite(events, database)
    begin
        @influxdbClient.write_points(events, @time_precision, @retention_policy, database )
    rescue InfluxDB::AuthenticationError => ae
        @logger.warn("Authentication Error while writing to InfluxDB", :exception => ae)
    rescue InfluxDB::ConnectionError => ce 
        @logger.warn("Connection Error while writing to InfluxDB", :exception => ce)
    rescue Exception => e
        @logger.warn("Non recoverable exception while writing to InfluxDB", :exception => e)
    end
  end

  # ...
~~~


## influxdb-ruby를 이용한 테스트
* github : https://github.com/influxdata/influxdb-ruby
* irb, ruby 2.3.3p222
~~~ruby
> require 'influxdb'
> influxdb = InfluxDB::Client.new host: "logis-dp-01"

> ## data type test
> database = 'ruby'
> time_precision = 'ms'
> retention_policy = 'autogen'

> data = [
  {
    series: 'cpu',
    values: { cpu1: 5, cpu2: 0, cpu3: 0.0}
  }
]

> influxdb.write_points(data, time_precision, retention_policy, database)

> data = [
  {
    series: 'cpu',
    values: { b02: TRUE}
  }
]

> influxdb.write_points(data, time_precision, retention_policy, database)
~~~

