---
title: 关于LogStash的一些小结
description: 一直在学，却总是学不会
date: 2021-05-29
slug: elk/logstash/basic
image: img/2021/05/miku.jpg
categories:
  - elk
---

虽然早在不久前就已经开发了一个日志采集的组件并学习了 ELK 的部署方案，但事实证明，不下功夫去理清细节的话，每次用的时候都要重新再学一遍。

之前虽然开发的差不多了，但是一直没有用到；然后本周，终于开始和其他端联调了，发现了很多不足的地方，于是又花了不少功夫学习，所以记录一下以防又忘记了。

## 简单介绍

日志组件，主要负责接收用户端传递的日志文件，然后丢到 Kafka，再经过 LogStash，最后到达 ES。服务端的日志，统一采用 FileBeat 进行采集。

## FileBeat 的使用

FileBeat 的配置相对来说比较简单，配置好 input 和 output 就完事了。官方文档中有很多种配置条件，只不过基本上都跟场景密切关联的，下面是我用到的一些配置。

```yml
filebeat.inputs:
  - type: log # 最常用的类型，抽取log文件的内容
    enabled: true # 默认为true，可以不配置这行
    paths: # 日志的具体路径
      - /var/log/*.log
    recursive_glob:
      enable: false # 是否解析`**` => 递归查找日志文件
    include_lines: ["^WARN", "^ERROR"] # 抽取匹配成功的日志
    exclude_lines: ["^DEBUG"] # 通过正则匹配，不抽取匹配成功的日志
    fields: # 通过配置fields, 可以给当前日志添加上该属性
      sampleKey: "sampleValue"
output.logstash: # 配置好输出方的地址即可，这里采用LogStash是为了对日志进行统一处理
  hosts: ["logstash:5044"]
```

## LogStash 的配置

LogStash 支持许多种输入、输出的方案，本次开发过程中仅用到了 FileBeat 和 Kafka 输入，ES 输出。详细配置如下：

```yml
input {
  beats {
      # 添加属性Type，用于后续区分处理
      type => "filebeat"
      port => "5044"
  }

  kafka {
      type => "kafka"
      bootstrap_servers => "kafka:9092"
      topics => ["default-topic"] # 配置订阅的主题
      codec => json
  }
}

filter {
  # 通过type区分日志来源
  if [type] == "filebeat" {
    # 采用grok插件对日志进行解析
    grok {
      match => [
        # 这条规则主要用来解析SpringBoot的日志(logback框架)
        "message", "(?<timestamp>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}.\d{3}) \[%{NOTSPACE:thread}\] %{NOTSPACE:level} %{NOTSPACE:class} - %{GREEDYDATA:msg}"
      ]
    }
    # 如果匹配失败，则直接丢掉
    if "_grokparsefailure" in [tags] {
      drop {}
    }
    # 通过date插件，将当前日志中的时间提取出来，并替换掉`@timestamp`
    date {
      match => ["timestamp", "yyyy-MM-dd HH:mm:ss.SSS"]
      target => "@timestamp" # 默认配置，可省略
    }
  }

  if [type] == "kafka" {
    # 采用json插件，对kafka采集的日志进行解析
    json {
      source => "message"
    }
  }
}

output {
  if [type] == "filebeat" {
      elasticsearch {
          hosts => ["http://elasticsearch:9200"]
          # 自定义index，在这里可以获取到在FileBeats中额外添加的字段[fields.sampleKey]
          index => "%{[fields][sampleKey]}-%{+YYYY.MM.dd}"
      }
  }

  if [type] == "kafka" {
      elasticsearch {
          hosts => ["http://elasticsearch:9200"]
          # 由于经过json解析，可以直接获取到对象的内容
          index => "%{key}-%{+YYYY.MM.dd}"
      }
  }
}
```

## 最后

grok 的脚本可以在[这里](https://grokdebug.herokuapp.com/)测试。

怎么说呢，感觉还是不太行，连会用都算不上，暂时先这样吧，有时间了系统学习一下 ELK。

## 参考链接

- [FileBeat Input](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-log.html)
- [LogStash Filter Plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)
