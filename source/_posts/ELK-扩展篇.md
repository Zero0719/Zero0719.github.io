---
title: ELK 扩展篇
date: 2024-04-09 22:20:16
tags: elk
---

为什么需要用 `filebeat`, `kafka` 扩展 `ELK` ，看看 `chatgpt` 的回答:

> 1. **负载均衡和缓冲**: Filebeat负责将日志数据发送到Kafka中间件，而Kafka充当缓冲层，能够平滑处理突发的数据流量和高峰时段的负载。这种负载均衡和缓冲机制可以有效地减少直接将日志数据发送到Elasticsearch时可能出现的压力和风险。
> 2. **数据可靠性和持久性**: Kafka提供了高度可靠的消息传递机制，能够保证数据不会丢失，并且支持数据持久化存储。这意味着即使Elasticsearch出现故障或需要维护时，数据仍然安全存储在Kafka中，不会丢失
> 3. **解耦**: 将Filebeat和Kafka引入ELK系统中，可以实现解耦，即日志收集与日志处理之间的解耦。这意味着系统的不同部分可以独立扩展和调整，而不会影响整个系统的稳定性
> 4. **灵活性**: 使用Kafka作为中间件，可以实现灵活的数据流动和处理。你可以根据需求调整Filebeat和Kafka的配置，以满足不同的性能和可用性要求

这里基于上一篇文章 {% post_link ELK-搭建 %} 的基础上，我们加入 `kafka` 和 `filebeat`

<!-- more -->

此时我们的架构如下：
`filebeat` -> `kafka` -> `logstash` -> `elasticsearch` -> `kibana`

**注意，我在前面基础篇的时候机子的性能是 2cpus，4g内存，在进行这篇文章所有服务启动时会导致机器卡死，所以后面调整为了 4cpus，8g内存**

## kafka 的安装

[文档](https://kafka.apache.org/quickstart)

```bash
wget https://dlcdn.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
tar -xzvf kafka_2.13-3.7.0.tgz
cd kafka_2.13-3.7.0/

# 注意，kafka 的使用需要安装 java 环境，sudo apt install openjdk-11-jdk
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
bin/kafka-server-start.sh -daemon config/server.properties

# 查看服务是否已经启动
ps aux | grep kafka

# 新建主题
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 5 --topic elk-fk

# 控制台监听主题，测试完毕以后，这个终端保持先不关
bin/kafka-console-consumer.sh --topic elk-fk --bootstrap-server localhost:9092 

# 另起一个终端产生消息，测试完毕可退出这个终端
bin/kafka-console-producer.sh --topic elk-fk --bootstrap-server localhost:9092
```

![](/images/ELK-扩展/1.png)
![](/images/ELK-扩展/2.png)

## filebeat 的安装

[文档]()

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.13.2-linux-x86_64.tar.gz
tar xzvf filebeat-8.13.2-linux-x86_64.tar.gz
cd filebeat-8.13.2-linux-x86_64/
```

先不启动 `filebeat`，我们目前需要考虑的是 `filebeat` 需要监听什么地方的数据，以及怎么样的把数据投递到 `kafka` 中


```bash
# test_log 为我们需要监听的目录
mkdir /home/vagrant/test_log

# 修改 filebeat 的配置文件 filebeat.yml
vim filebeat.yml
```
```yaml
###################### Filebeat Configuration Example #########################

# This file is an example configuration file highlighting only the most common
# options. The filebeat.reference.yml file from the same directory contains all the
# supported options with more comments. You can use it as a reference.
#
# You can find the full configuration reference here:
# https://www.elastic.co/guide/en/beats/filebeat/index.html

# For more available modules and options, please see the filebeat.reference.yml sample
# configuration file.

# ============================== Filebeat inputs ===============================

filebeat.inputs:

# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.
# Below are the input-specific configurations.

# filestream is an input for collecting log messages from files.
#- type: filestream

  # Unique ID among all inputs, an ID is required.
  #id: my-filestream-id

  # Change to true to enable this input configuration.
  #enabled: false

  # Paths that should be crawled and fetched. Glob based paths.
  #paths:
  #- /var/log/*.log
    #- c:\programdata\elasticsearch\logs\*
- type: log
  enabled: true
  paths:
    - /home/vagrant/test_log/*.log
  # Exclude lines. A list of regular expressions to match. It drops the lines that are
  # matching any regular expression from the list.
  # Line filtering happens after the parsers pipeline. If you would like to filter lines
  # before parsers, use include_message parser.
  #exclude_lines: ['^DBG']

  # Include lines. A list of regular expressions to match. It exports the lines that are
  # matching any regular expression from the list.
  # Line filtering happens after the parsers pipeline. If you would like to filter lines
  # before parsers, use include_message parser.
  #include_lines: ['^ERR', '^WARN']

  # Exclude files. A list of regular expressions to match. Filebeat drops the files that
  # are matching any regular expression from the list. By default, no files are dropped.
  #prospector.scanner.exclude_files: ['.gz$']

  # Optional additional fields. These fields can be freely picked
  # to add additional information to the crawled log files for filtering
  #fields:
  #  level: debug
  #  review: 1

# ============================== Filebeat modules ==============================

filebeat.config.modules:
  # Glob pattern for configuration loading
  path: ${path.config}/modules.d/*.yml

  # Set to true to enable config reloading
  reload.enabled: false

  # Period on which files under path should be checked for changes
  #reload.period: 10s

# ======================= Elasticsearch template setting =======================

setup.template.settings:
  index.number_of_shards: 1
  #index.codec: best_compression
  #_source.enabled: false


# ================================== General ===================================

# The name of the shipper that publishes the network data. It can be used to group
# all the transactions sent by a single shipper in the web interface.
#name:

# The tags of the shipper are included in their field with each
# transaction published.
#tags: ["service-X", "web-tier"]

# Optional fields that you can specify to add additional information to the
# output.
#fields:
#  env: staging

# ================================= Dashboards =================================
# These settings control loading the sample dashboards to the Kibana index. Loading
# the dashboards is disabled by default and can be enabled either by setting the
# options here or by using the `setup` command.
#setup.dashboards.enabled: false

# The URL from where to download the dashboard archive. By default, this URL
# has a value that is computed based on the Beat name and version. For released
# versions, this URL points to the dashboard archive on the artifacts.elastic.co
# website.
#setup.dashboards.url:

# =================================== Kibana ===================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"

  # Kibana Space ID
  # ID of the Kibana Space into which the dashboards should be loaded. By default,
  # the Default Space will be used.
  #space.id:

# =============================== Elastic Cloud ================================

# These settings simplify using Filebeat with the Elastic Cloud (https://cloud.elastic.co/).

# The cloud.id setting overwrites the `output.elasticsearch.hosts` and
# `setup.kibana.host` options.
# You can find the `cloud.id` in the Elastic Cloud web UI.
#cloud.id:

# The cloud.auth setting overwrites the `output.elasticsearch.username` and
# `output.elasticsearch.password` settings. The format is `<user>:<pass>`.
#cloud.auth:

# ================================== Outputs ===================================

# Configure what output to use when sending the data collected by the beat.

# ---------------------------- Elasticsearch Output ----------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["localhost:9200"]

  # Performance preset - one of "balanced", "throughput", "scale",
  # "latency", or "custom".
  #preset: balanced

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"

# ------------------------------ Logstash Output -------------------------------
#output.logstash:
  # The Logstash hosts
  #hosts: ["localhost:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

# ================================= Processors =================================
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

# ================================== Logging ===================================

# Sets log level. The default log level is info.
# Available log levels are: error, warning, info, debug
#logging.level: debug

# At debug level, you can selectively enable logging only for some components.
# To enable all selectors, use ["*"]. Examples of other selectors are "beat",
# "publisher", "service".
#logging.selectors: ["*"]

# ============================= X-Pack Monitoring ==============================
# Filebeat can export internal metrics to a central Elasticsearch monitoring
# cluster.  This requires xpack monitoring to be enabled in Elasticsearch.  The
# reporting is disabled by default.

# Set to true to enable the monitoring reporter.
#monitoring.enabled: false

# Sets the UUID of the Elasticsearch cluster under which monitoring data for this
# Filebeat instance will appear in the Stack Monitoring UI. If output.elasticsearch
# is enabled, the UUID is derived from the Elasticsearch cluster referenced by output.elasticsearch.
#monitoring.cluster_uuid:

# Uncomment to send the metrics to Elasticsearch. Most settings from the
# Elasticsearch outputs are accepted here as well.
# Note that the settings should point to your Elasticsearch *monitoring* cluster.
# Any setting that is not set is automatically inherited from the Elasticsearch
# output configuration, so if you have the Elasticsearch output configured such
# that it is pointing to your Elasticsearch monitoring cluster, you can simply
# uncomment the following line.
#monitoring.elasticsearch:

# ============================== Instrumentation ===============================

# Instrumentation support for the filebeat.
#instrumentation:
    # Set to true to enable instrumentation of filebeat.
    #enabled: false

    # Environment in which filebeat is running on (eg: staging, production, etc.)
    #environment: ""

    # APM Server hosts to report instrumentation results to.
    #hosts:
    #  - http://localhost:8200

    # API Key for the APM Server(s).
    # If api_key is set then secret_token will be ignored.
    #api_key:

    # Secret token for the APM Server(s).
    #secret_token:


# ================================= Migration ==================================

# This allows to enable 6.7 migration aliases
#migration.6_to_7.enabled: true
output.kafka:
  hosts: ["localhost:9092"]
  topic: "elk-fk"
```

启动 `filebeat`

```bash
./filebeat -c filebeat.yml
```

```bash
# 插入内容
echo 'hi' >> 1.log
echo 'hi' >> 1.log
echo 'hi' >> 1.log
echo 'hi' >> 1.log
echo 'hi' >> 1.log
echo 'hi' >> 1.log
echo 'hi' >> 1.log
```

我们在之前保留的监听终端可以看到相关的输出

![](/images/ELK-扩展/3.png)

## 关联 logstash 与 kafka 

修改 `logstash.conf`
```
cd logstash-8.13.1/
vim config/logstash.conf

# 修改前
-----------------
input {
  file {
    path => "/home/vagrant/test.log"
    start_position => "beginning"
  }
}

# 修改后
-----------------
input {
  kafka {
    bootstrap_servers => "localhost:9092"
    topics => ["elk-fk"]
    group_id => "elk-fk-group"
  }
}
```

输入内容
```bash
echo 2222222222222 >> 2.log
echo 33333333333333 >> 3.log
echo "are you ok" >> 1.log
```
![](/images/ELK-扩展/4.png)

刚好对应 3 条消息
![](/images/ELK-扩展/5.png)

大功告成啦！！！一个企业级的日志系统就部署好了，当然还有很多细节上的东西可以打磨调整，后续想到再写吧
