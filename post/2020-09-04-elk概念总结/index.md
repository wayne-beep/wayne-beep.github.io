
# Search查询

## 1. 搜索

1. 查询索引

   ```shell
   GET /_search  #查询所有索引信息
   GET /index1/_search #查询指定索引信息
   GET /index1,index2/_search #同时查询多个索引
   GET /index*/_search #通配符匹配索引
   ```

## 2. URI查询

```shell
GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s
{
		"profile": "true"
} 
```

- q 指定查询语句，使用Query String Syntax
- df 默认字段，不指定会对所有字段进行查询，这样查询效率低
- sort 排序
- from、size 用于分页
- profile 可以查看查询是如何被执行的

### Query String Syntax

1. 指定字段 VS 泛查询

   q=title:2012 VS q=2012 

2. Term查询 VS Phrase查询

   - Beautiful Mind 等效于Beautiful OR Mind

     ```shell
     GET /movies/_search?q=title: "Beautiful Mind"
     {
       "profile": "true"
     }
     ```

   - ”Beautiful Mind“，等效于 Beautiful AND Mind。Phrase查询还要求前后顺序保持一致

     ```shell
      GET /movies/_search?q=title: "Beautiful Mind"
     {
       "profile": "true"
     }
     ```

     

3. 分组与引号查询

   - title:(Beautiful AND Mind)
   - title="Beautiful Mind"

# Mapping & Dynamic Mapping

## 什么是Mapping？

 - Mapping类似数据库中的schema的定义，作用如下
   - 定义索引中的字段的名称
   - 定义字段的数据类型，例如字符串、数字、布尔……
   - 字段，倒排索引的相关位置（Analyzed or Not Analyzed, Analyzer)
 - Mapping会把JSON文档映射成Lucene所需要的扁平格式
 - 一个Mapping属于一个索引的Type
   - 每个文档都属于一个Type
   - 一个Type有一个Mapping定义
   - 7.0开始，不需要在Mapping定义中指定Type信息

#### 字段的数据类型

- 简单类型
  1. Text / Keyword
  2. Date
  3. Integer / Floating
  4. Boolean
  5. IPv4 & IPv6
- 复杂类型 - 对象和嵌套对象
  - 对象类型 嵌套类型
- 特殊类型
  - geo_point & geo_shape / percolator

## Dynamic Mapping

在写入文档的时候，如果索引不存在，会自动创建索引，Dynamic Mapping的机制使得我们无需手动定义Mappings。Elasticsearch会根据文档信息推算出字段的类型。但是有的时候推算的不准确，例如地理位置信息

| JSON类型  | elasticsearch类型  |
| ---- | ------ |
| 字符串 | 1. 匹配日期格式，设置成Date<br>2. 配置数字设置为float或者long，该选项默认关闭<br>3. 设置为Text，并且增加keyword子字段 |
| 布尔值 | boolean |
| 浮点数 | float |
| 整数     | long |
| 对象     | Object |
| 数组     | 由第一个非空数值的类型所定义 |
| 空值     | 忽略 |

例：

```json
# kibana dev tool
PUT mapping_test/_doc/1
{
  "uid": "123",
  "isVip": false,
  "isAdmin": "true",
  "age": 19,
  "loginDate": "2020-07-24T10:29:48.103Z"
}
# 返回
{
  "_index" : "mapping_test",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

GET mapping_test/_mapping
{
  "mapping_test" : {
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "long"   #年龄匹配成整型
        },
        "firstName" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "isAdmin" : {
          "type" : "text",  #由于true包含双引号，因此匹配成text类型，同时加上了keyword
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "isVip" : {
          "type" : "boolean"  # false匹配成布尔值
        },
        "lastName" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "loginDate" : {
          "type" : "date"  #日期格式匹配成date 
        },
        "uid" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
```

#### 能否更改Mapping的字段类型

1. 新增字段
   - Dynamic设置为true时，一旦有新增的文档写入，Mapping也同时被更新
   - Dynamic设置为false时，Mapping不会被更新，新增字段的数据无法被索引，但信息会出现在_source中
   - Dynamic设置为Strict，文档写入失败
2. 对已有字段，一旦已经有数据写入，就不再支持修改字段定义
	Lucene实现的倒排索引，一旦生成后，就不允许修改，如果希望改变字段类型，必须Reindex API重建索引，或者索引在每日轮转后生成新的所有名而生效。

原因

- 如果修改了字段的数据类型，会导致已经被索引的属于无法被搜索
- 如果是增加新的字段就不会有这样的影响

#### 控制Dynamic Mappings

|                | true | false | strict |
| -------------- | ---- | ----- | ------ |
| 文档可索引     | Y    | Y     | N      |
| 新增字段可索引 | Y    | N     | N      |
| Mapping被更新  | Y    | N     | N      |

```json
PUT movies
{
  "mappings": {
    "_docs": {
      "dynamic": "false"
    }
  }
}
```

#### 精确值（Exact Values）VS 全文本（Full Text）

- Exact Values精确值包括数字、日期、具体一个字符串（elasticsearch中的keyword），不会被分词
- 全文本，非结构化的文本数据（elasticsearch中的text）可以分词

  

## Index Template 和 Dynamic Template

### Index Template

index template帮助我们设定Mappings和Settings ，并按照一定的规则，自动匹配到新创建的索引之上

- 模板仅在一个索引被创建时才会产生作用。修改模板不会影响已创建的索引
- 可以设定多个索引模板，这些设置可以merge在一起
- 可以指定“order”的数值来控制merge的过程

#### Index Template的工作方式

当一个索引被创建时

- 应用Elasticsearch默认的settings和Mappings
- 应用order数值低的index template中的设定
- 应用order高的index template中的设定，之前的设定会被覆盖
- 应用创建索引时，用户所指定的Settings和Mappings，并覆盖之前模板中的设定

## Elasticsearch聚合Aggregation

1. Elasticsearch除了搜索以外，提供针对ES数据进行统计分析的功能
2. 通过聚合，我们会得到一个数据的概览，是分析总结全套的数据，而不是寻找单个文档
3. 高性能，只需要一条语句，就可以从Elasticsearch得到分析结果，无需在客户端自己去实现分析逻辑

### 集合的分类

- Bucket Aggregation 一些列满足特定条件的文档的集合
- Metric Aggregation 一些数学运算，可以对文档字段进行统计分析
- Pipeline Aggregation 对其他的聚合结果进行二次聚合
- Matrix Aggregation支持对多个字段的操作并提供一个结果的矩阵

#### Bucket & Metric

- bucket 一组满足条件的文档，相当于mysql的 select count(bound)
- metric 一些系列的统计方法，相当于mysql的group by brand

 ## 测试题

1. ES支持使用HTTP PUT写入新文档，并通过ES生成文档ID

   答：错，只能用POST方式写入

2. Update一个文档，需要使用HTTP PUT

   答：错，使用POST方式，PUT只能进行index和create

3. ES中的analyzer有哪几部分组成的

   答：character Filter + Tokenizer + Token Filter

4. 描述match和match_phrase的区别

   答：match 中的terms直接的or关系，Match Phrase的terms之间是and关系，并且term之间的位置关系也影响到搜索的结果

5. 如果你希望match_phrase匹配到更多的结果，应该配置查询中的什么参数

   答：slop参数

6. 判断：可以把一个字段类型从integer改成long，因为两个类型是兼容的

   错，字段类型的修改，在有数据进来之后，是需要重新index的

7. 判断：可以在Mapping文件中为index和searching指定不通的analyzer

   对

8. 判断： 字段类型为Text的字段一定可以被全文搜索

   错，可以通过为text类型的字段类型指定为Not Indexed，使其无法被搜索

# 运维

## 集群身份认证与用户鉴权

### ES信息出现泄漏的原因分析

- Elasticsearch在默认安装后，不提供任何形式的安全防护

- 错误的配置信息导致公网可以访问ES集群

  例如server.host被错误的配置成0.0.0.0

### 数据安全性的基本需求

1. 身份认证

   鉴定用户是否合法

2. 用户鉴权

   指定哪个用户可以访问哪个索引

3. 传输加密

4. 日志审计

#### 免费的安全方案

1. 设置nginx反向代理
2. 安装免费的Security插件
3. x-pack

### Authentication 身份认证

#### 认证体系的几种类型

1. 提供用户名密码
2. 提供秘钥或Kerberos票据

#### Realms：X-Pack中的认证服务

1. 内置Realms（免费）

   File/Native（用户名密码保存在ES）

2. 外部Realms（收费）

   Ldap/AD/PKI/SAML/Kerberos

### RBAC 用户鉴权

什么是RBAC：Role Based Access Control ，定义一个角色并分配一组权限。权限包括索引级，字段级，集群级的不同操作。然后通过将角色分配给用户，使得用户拥有这些权限。

#### 集群内部传输加密

为节点创建证书

TLS：TLS协议要求Trust Certificate Authority（CA）签发的X.509的证书

证书认证的不同级别

1. Certificate 节点加入需要使用相同CA签发的证书
2. Full Verfication 节点加入集群需要相同CA签发的证书，还需要Host name 活IP地址
3. No Verfication 任何节点都可以加入，开发环境中用于诊断目的

#### 集群与外部安全加密

对外传输加密使用https的方式

1. 配置Elasticsearch集群直接的https，ES集群每个节点开启如下参数

   ```yaml
   xpack.security.http.ssl.enabled: true
   xpack.security.http.ssl.keystore.path: /etc/elasticsearch/certs/elastic-certificates.p12 # 之前生成的证书
   xpack.security.http.ssl.truststore.path: /etc/elasticsearch/certs/elastic-certificates.p12
   ```

   然后重启集群

2. 配置kibana连接ES

   ES集群使用https访问方式后，kibana也需要进行设置

   ```yaml
   elasticsearch.hosts: ["https://10.40.98.100:9200"]  # 这里由http改成https
   elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/certs/elastic-ca.pem" ] #这个证书需要单独生成
   elasticsearch.ssl.verificationMode: certificate
   ```

   由于kibana的certificateAuthorities证书不能直接使用Elasticsearch的elastic-certificates.p12，要使用pem格式证书，，因此需要使用openssl工具将elastic-certificates.p12导出证书`openssl pkcs12 -in elastic-certificates.p12 -cacerts -nokeys -out elastic-ca.pem` 

   修改完成后重启kibana

   

## 部署方式

#### 节点参数配置

| 节点类型          | 配置参数    | 默认值                    | 配置                                                         |
| ----------------- | ----------- | ------------------------- | ------------------------------------------------------------ |
| master eligible   | node.master | true                      | n\ode.master: true<br>node.ingest: false<br>node.data: false |
| data              | node.data   | true                      | node.master: false<br/>node.ingest: false<br/>node.data: true |
| ingest            | node.ingest | true                      | node.master: false<br/>node.ingest: true<br/>node.data: false |
| coordinating only | -           | 设置上面三个参数全部false | node.master: false<br/>node.ingest: false<br/>node.data: false |
| machine learning  | node.ml     | True(需要enable x-pack)   | node.master: false<br/>node.ingest: false<br/>node.data: false<br>node.ml: true |

#### 单一角色职责分离的好处

- Dedicated master eligible nodes: 负责集群状态的管理，使用低配的CPU、RAM和磁盘即可
- Dedicated data nodes: 负责数据存储及处理客户端请求，使用高配置的CPU、RAM和磁盘
- Dedicated ingest nodes: 负责数据处理，使用高配置CPU、中等配置RAM、低配置的磁盘
- Dedicated coordinating only nodes (client noly): 中高CPU、中高RAM、低磁盘

生产环境中、建议为一些大的集群配置Coordinating Only Nodes，它们可以扮演LB，降低master和data node的负载。负责搜索结果的聚合、分解。有时候无法预知客户端会发送怎么样的请求，大量的占用内存的结合操作，一个深度的聚合可能引发OOM。

#### 增加节点，水平扩展

当磁盘容量无法满足需求时，可以增加数据节点，磁盘读压力大时，增加数据节点。当系统中有大量的复杂查询及聚合的时候，增加Coordinating节点，增加查询的性能。

#### 读写分离

读使用Coordinating node，写可以通过增加ingest node，当写性能不足，可以扩展ingest node节点。

#### 部署kibana

官方建议，可以将kibana部署在Coordinating node上，前端也通过LB进行代理

## Hot & Warm架构与Shard Filtering

数据通常不会有Update操作，适用于Time based索引数据（生命周期管理），同时数据量比较大的场景，引入warm节点，低配置大容量的机器存放老数据以降低部署成本。

- Hot节点（通常使用SSD) ：索引不断新文档写入。通常使用SSD
- Warm节点（通常使用HDD）：索引不存在新数据的写入；同时也不存在大量的数据查询

#### 标记节点

需要通过“node.attr”来标记一个节点，节点的attribute可以是任何的key/value，例如elasticsearch.yaml中配置node.attr.box_type=hot  node.attr.box_type=warm。通过接口查看节点key value对`GET /_cat/nodeattrs?v`

#### 新索引分配到Hot节点

当新建索引的时候，可以通过索引模板的settings块内加入新索引配置分配到的节点

```json
	
	"index": {
      "routing": {
        "allocation": {
          "include": {
            "box_type": "hot"
          }
        }
      }
```

#### 就索引分配到warm节点

Index.routing.allocation是一个索引级的Dynamic setting，可以通过API后期进行设定

```json
PUT logstash-2020-09-05/_settings
{
  "index.routing.allocation.require.box_type": "warm"
}
```

Elasticsearch会自动将索引reallocate到warm节点当中去

## 多机架部署Rack Awareness

ES节点可能分配在不同的机架

- 当一个机架断电，可能会同时丢失几个节点
- 如果一个索引相同的主分片和副本分片同时在这个机架上，就有可能丢失数据
- 通过Rack Awareness机制，就可以尽可能避免将同一个索引的副本分片同时分配在一个机架的节点上

通过在配置文件中设置`node.attr.rack_id=rack1`来标记

```json
PUT _cluster/settings/
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "rack_id",
    "cluster.routing.allocation.awareness.force.zone.values": "rack1,rack2"
  }
}
PUT my_index1
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 2
  }
}
PUT my_index1/_doc/1
{
  "key": "value"
}

GET _cat/shards?v
```

## 分配设定与管理

分片Shared是ES集群实现集群水平扩展的最小单位。从7.0开始，新创建索引默认只有一个主分片，单个分片，查询算分聚合不准的问题都可以避免。单个索引单个分片的时候，集群无法实现水平扩展，即使增加节点。

### 如何设计分片数

1. 当分片数 > 节点数，一旦集群中有新节点加入，分片就可以自动进行分配，分片在重新分配时，系统不会有downtime
2. 多分片的好处是一个索引分片如果分布在不同的节点，多个节点可以并行执行查询、写入
3. 每个分片是一个Lucene的索引，会使用机器的资源。过多的分配数会带来一些潜在的问题
   - Lucene Indices / File descriptors / RAM / CPU
   - 每次搜索的请求，需要从每个分片上获取数据
   - 分片的meta信息由master节点维护，过多会增加管理负担 

#### 如何确定分片数

从存储的物理角度看

1. 日之类应用，单个分配不超过50G，我们是15G
2. 搜索类应用，单个分片不超过20G

控制分片存储大小的好处

1. 提高update性能
2. Merge时，减少所需的资源
3. 丢失节点后，具备更快的恢复速度
4. 便于在集群内Rebalancing

#### 如何确定副本分片数

副本是主分片的拷贝

- 提高系统可用性：处理相应查询请求，防止数据丢失
- 需要占用和主分片一样的资源

对性能的影响：

- 副本会降低数据索引的速度，有几份副本就会有几倍的CPU资源消耗在索引上
- 会减缓对主分片查询的压力，但是会消耗同样的内存资源。如果机器资源充足，提高副本数可以提高整体的QPS

#### 集群容量规划

一个集群有多少节点，一个索引需要多少分片？规划上要保持一定的冗余量，当负载出现波动，节点出现丢失时还能正常运行。

做容量规划需要考虑的因素

- 机器的软硬件配置
- 单条文档的尺寸、文档的总数据量、索引的总数据量（time base数据保留时间）、副本分片数
- 文档是如何写入的（Bulk尺寸）
- 文档的复杂度，文档是如何进行读取的（怎么样的查询和聚合）

#### 评估业务的性能要求

1. 数据吞吐及性能要求
   - 数据写入的吞吐量，每秒要求写入多少数据
   - 查询的吞吐量
   - 单条查询可接受的最大返回时间
2. 了解你的数据
   - 数据的格式和数据的Mapping
   - 实际的查询和聚合长的是什么样的

#### 写入时间序列的数据：基于Date Math的方式

1. 容易使用
2. 如果时间发生变化，需要重新部署代码

| 设定                  | 结果                     |
| --------------------- | ------------------------ |
| <logs-{now/d}>        | Logs-2020-09-07          |
| <logs-{now{YYYY.MM}}> | Logs-2020-09             |
| <logs-{now/w}         | Logs-2019.7.29 #星期开始 |

#### 集群扩容

如果查询聚合性能低，可以扩展Coordinating、Ingest Node

增加数据节点：解决存储问题；避免分片不均匀问题，提前监控磁盘空间，提前清理数据或增加节点（70%）

#### JVM设定

从ES6开始，只支持64位的JVM，配置文件 config/jvm.options

1. 将Xms和Xmx设置成一样，避免Heap resize时引发停顿

2. Xmx设置不要超过物理内存的50%，剩下的50%加入Lucene，增加全文检索功能；单个节点上，最大内存不要超过32G

   > https://www.elastic.io/blog/a-heap-of-trouble

3. 生产环境JVM必须使用Server模式

4. 关闭JVM Swapping

#### 集群配置的API设定

elasticsearch集群的设置分为静态和动态

1. 静态方式，通过elasticsearch.yml配置文件中写入参数的方式。静态配置文件尽量简洁，尽量只写入必备的参数，静态方式修改需要重启节点

2. 动态方式，动态方式可以使用API动态进行设定。动态设定会覆盖elasticsearch.yml的配置，有两种模式：

   1. Transient： 在集群重启后会丢失
   2. Persistent：在集群重启后不会丢失

   ```json
   {
     "persistent":{},
     "transient":{
       "indices.recovery.max_bytes_per_sec": "20mb"
     }
   }
   ```

优先级

| 优先级 | 设置                  |
| ------ | --------------------- |
| 1      | Transient Settings    |
| 2      | Persistent Settings   |
| 3      | Command-line Settings |
| 4      | Config file Settings  |

## 最佳实践

#### 内存设定计算实例

内存大小要根据Node需要存储的数据来计算，内存磁盘配比如下：

- 搜索类比例建议1：16
- 日志类：1：48-96之间

总数据量1T，设置一个副本=2T总数据量

- 如果搜索类的项目，每个节点磁盘31G(内存)*16=496G，加上预留空间。所以每个节点最多400G数据，至少要5个数据节点
- 如果哦是日志累项目，每个节点磁盘31G(内存)*50 = 1550GB，2个数据节点即可

#### 存储

- 推荐使用SSD，使用本地存储，避免使用网络存储
- 可以在本地使用多个path.data以支持使用多块磁盘
- ES本身提供了很好的HA机制，无需使用RAID
- 可以是warm节点使用Spinning Disk，但是需要关闭Concurrent Merges：`index.merge.scheduler.max_thread_count:1`
- Trim你的SSD `https://www.elastic.io/blog/is-your-elasticsearch-trimmed`

#### 集群设置：关闭Dynamic Indexes

动态模板会导致大量的字段引入而影响性能，因此考虑关闭Dynamic Indexes功能

```json
PUT _cluster/settings
{
  "persistent":{
    "action.auto_create_index": false
  }
}
```

或者通过模板设定白名单，指定哪些索引可以自动创建

```json
PUT _cluster/settings
{
  "persistent":{
    "action.auto_create_index": "logstash-*,kibana-*"
  }
}
```

#### 集群安全

为Elasticsearch和kibana配置安全功能

- 打开authorization & authorization
- 实现索引和字段级的安全控制

节点通信加密

Enable HTTPS

Audit logs

#### 集群监控

EastICsearch Stats 相关API

- Node stats： _nodes/status 
- Cluster stats: _cluster/stats
- Index stats: index_name/_stats_
- task： _cluster/pending_tasks,

> 监控工具：https://github.com/elastic/support-diagnostics

 /usr/bin/curator --config /etc/curator/curator.yml /etc/curator/action-delete-snapshot.yml

GET _cluster/allocation/explain 返回未分配shard的原因

#### 添加watcher监控报警

模板

```json
{
  "trigger": {
    "schedule": {
      "interval": "10m"
    }
  },
  "input": {
    "search": {
      "request": {
        "search_type": "query_then_fetch",
        "indices": [
          "logstash-monix-sms-prod-*"
        ],
        "rest_total_hits_as_int": true,
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "match": {
                    "level": "ERROR"
                  }
                },
                {
                  "range": {
                    "@timestamp": {
                      "gt": "now-10m",
                      "lte": "now"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "app_aggs": {
              "terms": {
                "field": "type",
                "size": 20
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 10
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "profile": "standard",
        "to": [
          	"yangyuzhuo@wecash.net",
            "mayixin@wecash.net",
            "kongdewen@wecash.net",
            "fangpengfei@wecash.net",
          	"wangxingmin@wecash.net"
        ],
        "subject": "Monix SMS Error Watcher",
        "body": {
          "text": "本次检查触发时间: (UTC){{ ctx.execution_time }}\n在最近十分钟内发现以下错误:\n\n共计:{{ ctx.payload.hits.total }}条,本邮件最多展示报错最多的前10个type\n\n{{#ctx.payload.aggregations.app_aggs.buckets}}type: {{key}}\ncount: {{doc_count}}\n\n{{/ctx.payload.aggregations.app_aggs.buckets}}"
        }
      }
    }
  }
}
```
#### 数据写入过程

- Refresh： 将文档先保存在Index buffer中，以refresh_interval为间隔时间（默认1s），定期清空buffer，生成segment，借助文件系统缓存的特性，将segment放在文件系统缓存中，并开放查询，艺体生搜索的实时性。增加refresh_interval数值可避免过于频繁的refresh而生成过多的segment文件，但是会降低搜索的实时性。增大敬佩配置参数indices.memory.index_buffer_size（默认10%），会强制refresh
- Translog：Segment没有写入磁盘，即便发生了宕机，重启后，数据也能恢复，ES6.0以后，默认配置每次请求都会落盘。可降低写磁盘的频率，但是会降低容灾能力：
  - index.translog.durability：默认是request，每个请求都落盘。设置成async，异步写入
  - index.translog.sync_interval：设置为60s，每分钟执行一次
  - index.translog.flush_threshod_size：默认是512mb，可以适当增大，当translog超过该值，会触发flush
- Flush：删除旧的translog，生产Segment并写入磁盘，更新commint point并写入磁盘。ES自动完成

#### 分片的设定

1. 副本在写入时设为0，完成后再增加
2. 合理设置主分片数，确保均匀分配在所有的数据节点上
   - index.routing.allocation.total_share_per_node：限定每个索引在每个节点上可分配的主分片数。5个节点的集群，索引有5个分片，1个副本，应该如何设置：（5+5）/ 5 = 2。生产环境要调大这个数字，避免节点下线分片无法正常迁移

#### 查询性能优化

1. 查询时避免开头使用正则，这样性能会非常不好
2. 优化分片
   - 避免over sharding：一个查询需要访问每一个分片，分片过多，会导致不必要的查询开销
   - 结合应用场景，控制单个分片的尺寸：search: 20G，logging 40G

#### 压力测试








