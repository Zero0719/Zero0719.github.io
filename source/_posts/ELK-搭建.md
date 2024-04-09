---
title: ELK 搭建
date: 2024-04-09 11:16:57
tags: elk
---

`ELK` 是指由 `elasticsearch`, `logstash`, `kibana` 组成的日志管理解决方案

* `elasticsearch` 是一个分布式的实时搜索和分析引擎，能够快速地存储、搜索和分析大量数据
* `losstash` 能够从多种来源收集日志数据，对数据进行处理和过滤，然后将数据发送到Elasticsearch等目标存储中
* `kibana` 提供了一个直观的Web界面，可以用来创建实时仪表板、图表和可视化，帮助用户更好地理解和分析日志数据

<!-- more -->

> 以下内容部署基于 `ubuntu20` 虚拟机

## Elasticsearch 安装

[安装文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/targz.html)

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-8.13.2-linux-x86_64.tar.gz.sha512 
tar -xzf elasticsearch-8.13.2-linux-x86_64.tar.gz
cd elasticsearch-8.13.2/ 
```

```bash
# 启动
./bin/elasticsearch

# temp password 
*aG_h5WIMAcYVizoSecZ

# kibana 访问 code
eyJ2ZXIiOiI4LjEzLjIiLCJhZHIiOlsiMTAuMC4yLjE1OjkyMDAiXSwiZmdyIjoiZTAwYzQwMDM4MjA4NjFiMTllZTE5NmM2NGVmOTRjNGUzYWI3ZDk4YTc3ZTM5MTNhMGNhZTg2NzcwZWI3MDVhYiIsImtleSI6IldqSDF3STRCd2tfZl9hOWhhd3RHOm4tMmRzbVpqUk9tYkdMZHhOM1hLUncifQ==
```

测试
![测试](/images/ELK-搭建/1.png)

## Kibana 安装

[安装文档](https://www.elastic.co/guide/en/kibana/current/install.html)

```bash
curl -O https://artifacts.elastic.co/downloads/kibana/kibana-8.13.2-linux-x86_64.tar.gz
curl https://artifacts.elastic.co/downloads/kibana/kibana-8.13.2-linux-x86_64.tar.gz.sha512 | shasum -a 512 -c - 
tar -xzf kibana-8.13.2-linux-x86_64.tar.gz
cd kibana-8.13.2/
```

修改 `kibana` 配置文件
```bash
vim config/kibana.yml

------------------------------
# 因为我是部署在虚拟机上，所以这里要改为 0.0.0.0 可以让外部访问，同时虚拟机的配置需要允许访问 5601 端口
server.host: "0.0.0.0"

# 启动
./bin/kibana
```

![部署](/images/ELK-搭建/2.png)

输入上面保存的 `es` 可以让 `kibana` 访问的 `code`

![成功访问](/images/ELK-搭建/3.png)

## Logstash 安装

[安装文档](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html#package-repositories)

```bash
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.13.1-linux-x86_64.tar.gz
tar -xzvf logstash-8.13.1-linux-x86_64.tar.gz
cd cd logstash-8.13.1/
```

启动之前我们需要配置 `logstash` 监听某个文件或者目录，并传输到 `es` 中

```
# 假设该文件是我们要监听
touch /home/vagrant/test.log

# 修改 logstash 的配置文件 (如果该文件不存在，可以复制 logstash-sample.conf)
vim config/logstash.conf

--------------------

input {
  file {
    path => "/home/vagrant/test.log"
    start_position => "beginning"
  }
}

# 这里需要注意 es 的连接协议方式，信任token，部署的过程中对于这方面不熟悉，折腾了许久，后面是参考 kibana 中的配置文件才连接成功
output {
  elasticsearch {
    hosts => ["https://10.0.2.15:9200"]
    index => "test-elk"
    user => "elastic"
    password => "*aG_h5WIMAcYVizoSecZ"
    ca_trusted_fingerprint => "e00c4003820861b19ee196c64ef94c4e3ab7d98a77e3913a0cae86770eb705ab"
  }
}

```
![成功](/images/ELK-搭建/4.png)

## 在 Kibana 中创建 index 视图

![创建index视图](/images/ELK-搭建/5.png)

```bash
echo "hello elk" >> /home/vagrant/test.log
```

![](/images/ELK-搭建/6.png)

可以看见文件的变化已经被监听到写入到 `es` 中，`logstash` 还可以监听比如系统日志，`web` 日志，`nginx` 日志，等等你需要观察的日志记录，只需要增加对应的配置信息即可，至此，简单的 `elk` 搭建就已经完毕，但是这个架构方案存在一定问题，一是文件监听变化的 `logstash` 相对体量比较大，比较吃资源，二是如果 `logstash` 发生故障会导致一部分变化数据丢失，另一篇文章，我们在此架构基础上加上 `filebeat`, `kafka` 解决这两个问题，让整个日志系统更加稳定 





