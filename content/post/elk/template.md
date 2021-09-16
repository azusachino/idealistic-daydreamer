---
title: Logstash配置Template的泥泞之旅
description: 坑是踩不完的
date: 2021-06-03
slug: elk/logstash/template
image: img/2021/06/AsianElephants.jpg
categories:
  - elk
---

在最近联调的过程中，发现日志跑通整个链路的速度变得越来越慢，甚至延迟了几分钟才能在 Kibana 看到采集的日志信息，于是这几天花了很多时间排查问题。

通过打印时间戳，发现 Spring 读取小文件和 KafkaTemplate 的效率都很高，1M 左右的日志文件 300ms 左右就处理完了，然后大致猜测是 Logstash 的 Filter 和 Output 拉低了整体的吞吐量。

## 进行改善

### logstash 参数

根据网上的很多博客以及官方文档中的[Logstash settings file](https://www.elastic.co/guide/en/logstash/current/logstash-settings-file.html)，发现需要根据机器的配置适当调整批处理的规模，简单示例如下：

```yml
# logstash.yml
node:
  name: yourNodeName
pipeline:
  workers: 12 # 默认是`java.lang.Runtime.getRuntime.availableProcessors, 也就是CPU核心数
  batch:
    size: 300 # LogStash中input和output的BatchSize
    delay: 10 # BatchProcess interval
```

### 通过配置 Template 精确 Index

未配置 Template 的时候，es 索引的所有字段都是`text`，而且附带`fields.keyword`，这并不是我想要的结果。

通过参考网上的资料，发现最主要的办法就是针对`Index`定义`template`。

#### 开始动手

最开始的时候，根据 Kibana 中旧数据的`settings`和`mappings`，在本地组合成了 template，但是一旦重启 Logstash 就会出现错误`Failed to install template” / “ Got response code '400'`，而且错误信息中完全没有任何能参考的内容，很头疼。

#### 有所进展

通过查看 ES 的文档，大致总结出了下面的几个步骤(REST API)：

1. 访问 `elasticsearch:9200/_cat/templates` 查看 ES 中的模板列表
2. 通过 `elasticsearch:9200/_template/templateName` 获取特定 templateName 的配置
3. 下载 `logstash`，然后可以在其基础上进行修改
4. 通过 `PUT elasticsearch:9200/_template/yourTemplateName` 添加自定义的 template

另外就是，我们可以通过 PUT 方法校验`template`的是否合法，我的问题出在使用了未安装的分词器以及多了一层`_doc`。

#### 最终成果

下面给出一个采集 spring 程序日志的配置。

**pipelines.yml**:

```yml
- pipeline.id: spring-collector
  config.string: |
    input { 
      # filebeat采集日志
      beats {
        port => "5044"
      }
    }
    filter {
      # grok匹配 => json
      grok {
        match => [
          "message", "(?<timestamp>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}.\d{3}) \[%{NOTSPACE:thread}\] %{NOTSPACE:level} %{NOTSPACE:class} - %{GREEDYDATA:msg}"
        ]
      }
      # 如果匹配失败，直接丢掉
      if "_grokparsefailure" in [tags] {
        drop {}
      }
      # 指定记录的时间戳字段为采集时间
      date {
        match => [ "timestamp", "yyyy-MM-dd HH:mm:ss.SSS", "ISO8601" ]
        target => "@timestamp"
      }
      # 删除无用字段，减少带宽消耗
      mutate {
        remove_field => [ "timestamp", "message" ]
      }  
    }
    output {           
      elasticsearch {
        hosts => ["http://elasticsearch:9200"]
        index => "spring-%{+YYYY.MM.dd}" # 需要与模板中的index_patterns匹配
        template_name => "spring" # es/_cat/templates可以查询到的templateName
        template_overwrite => true
        template => "/usr/share/logstash/template/spring.json" # 模板文件
      }
    }
```

**spring.json**:

对 msg 字段采用了 ik_max_word 和 ik_smart 分词器，地址：[elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)，同时配置了`dynamic_templates`，即使出现了未知的字段也不会报错。

插件的安装很简单，下载和 ES 相同版本的包，然后解压到`usr/share/elasticsearch/plugins`中，然后重启 ES 即可。

```json
{
  "order": 0,
  "version": 60001,
  "index_patterns": ["spring-*"],
  "settings": {
    "index": {
      "number_of_shards": "1",
      "number_of_replicas": "0",
      "refresh_interval": "60s",
      "translog": {
        "durability": "async",
        "sync_interval": "60s"
      }
    }
  },
  "mappings": {
    "dynamic_templates": [
      {
        "message_field": {
          "path_match": "message",
          "mapping": {
            "norms": false,
            "type": "text"
          },
          "match_mapping_type": "string"
        }
      },
      {
        "string_fields": {
          "mapping": {
            "norms": false,
            "type": "keyword"
          },
          "match_mapping_type": "string",
          "match": "*"
        }
      }
    ],
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "@version": {
        "type": "keyword"
      },
      "agent": {
        "properties": {
          "ephemeral_id": {
            "type": "keyword"
          },
          "hostname": {
            "type": "keyword"
          },
          "id": {
            "type": "keyword"
          },
          "name": {
            "type": "keyword"
          },
          "type": {
            "type": "keyword"
          },
          "version": {
            "type": "keyword"
          }
        }
      },
      "class": {
        "type": "keyword"
      },
      "container": {
        "properties": {
          "id": {
            "type": "keyword"
          }
        }
      },
      "ecs": {
        "properties": {
          "version": {
            "type": "keyword"
          }
        }
      },
      "fields": {
        "properties": {
          "model": {
            "type": "keyword"
          }
        }
      },
      "host": {
        "properties": {
          "architecture": {
            "type": "keyword"
          },
          "containerized": {
            "type": "boolean"
          },
          "hostname": {
            "type": "keyword"
          },
          "id": {
            "type": "keyword"
          },
          "ip": {
            "type": "keyword"
          },
          "mac": {
            "type": "keyword"
          },
          "name": {
            "type": "keyword"
          },
          "os": {
            "properties": {
              "codename": {
                "type": "keyword"
              },
              "family": {
                "type": "keyword"
              },
              "kernel": {
                "type": "keyword"
              },
              "name": {
                "type": "keyword"
              },
              "platform": {
                "type": "keyword"
              },
              "type": {
                "type": "keyword"
              },
              "version": {
                "type": "keyword"
              }
            }
          }
        }
      },
      "input": {
        "properties": {
          "type": {
            "type": "keyword"
          }
        }
      },
      "level": {
        "type": "keyword"
      },
      "log": {
        "properties": {
          "file": {
            "properties": {
              "path": {
                "type": "keyword"
              }
            }
          },
          "offset": {
            "type": "long"
          }
        }
      },
      "msg": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "tags": {
        "type": "keyword"
      },
      "thread": {
        "type": "keyword"
      }
    }
  },
  "aliases": {}
}
```

## 简单总结

如果能吃透官方文档的话，问题应该早就迎刃而解了。

## 参考链接

- [Elasticsearch 中 mapping 全解实战](https://www.cnblogs.com/haixiang/p/12040272.html)
- [Index Templates](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/indices-templates.html)
