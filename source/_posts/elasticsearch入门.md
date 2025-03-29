---
title: elasticsearch入门
date: 2022-7-16 16:10:00
categories:
- 运维
---
## 简介

**从多个来源输入到 ES 中，数据在 ES 中进行索引和解析，标准化并充实这些数据。这些数据在 ES 中索引完成之后，用户就可以针对他们的数据进行复杂的查询，并使用聚合来检索这些数据，**

## 补充的docker命令

**删除所有状态为退出的容器：**

```
docker rm $(docker ps -a -f status=exited -q)
```

## 启动单节点集群

**为Elasticsearch和Kibana创建docker网络**

```
docker network create elastic
```

**启动**

```
docker run --name es01 --net elastic -p 9200:9200 -p 9300:9300 -it docker.elastic.co/elasticsearch/elasticsearch:8.2.2
```

**会生成elastic用户的密码和用于注册Kibana的token（有效期30min）**

```
->  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  xy1JLdqhhLIQemauzxdB

->  HTTP CA certificate SHA-256 fingerprint:
  fd16b3840cc28a25b8ba950e9bbc68f9a1fe84b538d2b8daf63bad27620041ef

->  Configure Kibana to use this cluster:
* Run Kibana and click the configuration link in the terminal when Kibana starts.
* Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjIuMiIsImFkciI6WyIxNzIuMTguMC4yOjkyMDAiXSwiZmdyIjoiZmQxNmIzODQwY2MyOGEyNWI4YmE5NTBlOWJiYzY4ZjlhMWZlODRiNTM4ZDJiOGRhZjYzYmFkMjc2MjAwNDFlZiIsImtleSI6ImhYNDBSNEVCUmptMkhGc3R6V2NFOmFUbjZURUJfUmhlUXhGYjh4bk9pVFEifQ==

-> Configure other nodes to join this cluster:
* Copy the following enrollment token and start new Elasticsearch nodes with `bin/elasticsearch --enrollment-token <token>` (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjIuMiIsImFkciI6WyIxNzIuMTguMC4yOjkyMDAiXSwiZmdyIjoiZmQxNmIzODQwY2MyOGEyNWI4YmE5NTBlOWJiYzY4ZjlhMWZlODRiNTM4ZDJiOGRhZjYzYmFkMjc2MjAwNDFlZiIsImtleSI6ImgzNDBSNEVCUmptMkhGc3R6V2NfOkw1ZG85U3d5UzN1TUJVd2VLdGtWTXcifQ==
```

**从docker容器中将http_ca.crt复制到本机上**

```
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt .
```

**测试能否连接上ES集群**

```
curl --cacert http_ca.crt -u elastic https://localhost:9200
```

> **Docker 挂载数据卷的默认权限是可读写(rw)，用户也可以通过 ro 标记指定为只读：**
>
> ```
> $ sudo docker run -d -P –-name web -v /var/data:/opt/webdata:ro myimg/webapp python app.py
> ```
>
> **加了 :ro 之后，容器内挂载的数据卷内的数据就变成只读的了。**

## 启动kibana

```
docker run --name kib-01 --net elastic -p 5601:5601 docker.elastic.co/kibana/kibana:8.2.2
```

**会显示一个链接，点击该链接，比如访问**[http://localhost:5601/?code=052166](http://localhost:5601/?code=052166)

**登录的用户就写elastic**

### 持久化kibana的一些配置

```
docker run --name kib-01 --net elastic -p 5601:5601 -v $PWD/kibana.yml:/usr/share/kibana/config/kibana.yml docker.elastic.co/kibana/kibana:8.2.2
```

## 如何写入数据

**目前 ELK 包含一系列丰富的轻量数据采集代理，这些代理被称之为 Beats。Beats将数据发送到Logstash或Elasticsearch。**

### 通过logstash同步文件

[一个简单的可用的ELK](https://zhuanlan.zhihu.com/p/158697278)

```
docker compose -f ELK.yml up -d # 启动
docker compose -f ELK.yml restart # 重启
```

**注意logstash.conf中需要指定，否则无法成功上报数据给ES：**

```
input {
  beats {
    port => 5044
  }
```

```
 elasticsearch {
    hosts => ["http://elasticsearch:9200"]  #elasticsearch请求地址
    index => "xxx"
```

**指定索引后，需要在kibana上创建索引，才可以成功查看。**

**关于 Logstash.conf的一些字段含义（语法），可以参考**[Logstash：解析 JSON 文件并导入到 Elasticsearch 中](https://blog.csdn.net/UbuntuTouch/article/details/114383426)

### 直接写ES

**对于go来说，使用官方client即可。**

```
es, err := elasticsearch.NewClient(elasticsearch.Config{
Addresses: []string{"http://127.0.0.1:9200"},
})
if err != nil {
log.Fatalf("Error creating the client: %s", err)
}
res, err := es.Info()
```

**注意客户端版本需要和服务端的版本一致。我使用的是** `github.com/elastic/go-elasticsearch/v7`

## logstash

**是一个数据分析软件，对数据进行聚合和处理，是一个开源的服务器端数据处理管道。**

**logstash是controller层，Elasticsearch是一个model层，kibana是view层。**

**首先将数据传给logstash，它将数据进行过滤和格式化（转成JSON格式），然后传给**[Elasticsearch](https://so.csdn.net/so/search?q=Elasticsearch&spm=1001.2101.3001.7020)进行存储、建搜索的索引，kibana提供前端的页面再进行搜索和图表可视化，它是调用Elasticsearch的接口返回的数据进行可视化。

### logstash.conf语法

**在生产环境中，Logstash管道会非常复杂：一般包含一个或多个输入，过滤器，输出插件。**

**logstash.conf语法主要就是关于logstash pipeline是如何写的。**

**pipeline工作内容： 从input读取事件源 ----> 经过filter解析和处理之后 ----> 从output输出到目标存储库（elasticsearch或其他）。**

[Logstash：解析 JSON 文件并导入到 Elasticsearch 中](https://blog.csdn.net/UbuntuTouch/article/details/114383426)

**使用 json codec中**，**sample.json**可以写多行，会解析成多个结果，也可以叫sample.log，需要在logstash.conf指定 `codec   => "json"`

**使用 JSON filter，filter里不需要指定** `codec   => "json"`了，但是要在filter里写 `json{source=>"message"}`

> **PS：需要多用，善用Dev Tools**
>
> **也可以请求**[http://127.0.0.1:9200](http://127.0.0.1:9200)来请求elasticsearch获取一些数据，比如GET [http://127.0.0.1:9200/_search](http://127.0.0.1:9200/_search)

## beats

** Filebeat 和 Logstash 都是请求ES的API来同步数据的。**

### Filebeat

**可以配置Filebeat发送日志到Logstash**

**Filebeat是轻量级，资源占用较低的日志收集工具，****从各服务器上收集日志并将这些日志发送到Logstash实例**中处理。

**Logstash默认安装已含有**[Beat input](https://link.juejin.cn?target=http%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Flogstash%2F6.5%2Fplugins-inputs-beats.html)插件。`Beat`输入插件使Logstash可以从 `Elastic Beat`框架接收事件，这意味者任何编写为与 `Beat`框架一起工作的Beat(如Packetbeat和Metricbeat)都可以将事件数据发送到Logstash。

### metricbeat 

**用于从系统和服务收集指标。Metricbeat 能够以一种轻量型的方式，输送各种系统和服务统计数据，从 CPU 到内存，从 Redis 到 Nginx，不一而足。**

**Metricbeat 提供多种内部模块，这些模块可从多项服务（Prometheus，Prometheus等）中收集指标。只需在配置文件中启用您所需的模块即可。除了这些服务，也可以构建自己的模块。**

### packetbeat

## 映射Mapping

**Mapping是用来定义一个文档（document），以及它所包含的属性（field）是如何存储和索引的。**

**比如：可以使用Mapping来定义哪些字符串属性应该被看作全文索引（full text fields），哪些属性包含数字，日期或者地理位置。文档中的所有属性是否都能被索引（_all 配置）。**

**在**[ES6](https://so.csdn.net/so/search?q=ES6&spm=1001.2101.3001.7020).0.0及更高的版本中，创建的索引只能包含一个映射类型。

**每一个映射类型包含：元标签，字段或属性。**

### 元数据

**在Elasticsearch下，一个文档除了有数据之外，它还包含了****元数据(Metadata)**。每创建一条数据时，都会对元数据进行写入等操作，当然有些元数据是在创建mapping的时候就会设置，元数据在Elasticsearch下起到了非常大的作用。

#### 身份元数据

**Identity meta-fields**

`_index`：文档所属的index，这个index相当于关系型数据库中的数据库概念，它是存储和索引关联数据的地方；

`_uid`：其由 `_type`和 `_id`组成；

`_type`：文档所属的mapping type，相当于关系型数据库中的**表**的概念；

`_id`：文档的id，这个可以由Elasticsearch自动生成，也可以在写入Document的时候由程序指定。它与 `_index`和 `_type`组合时，就可以在Elasticsearch中唯一标识一个文档。

#### 文档元数据

**Document source meta-fields**

`_source`：这个字段标识文档的主体信息，也就是我们写入在[ElasticSearch](https://www.iteblog.com/archives/tag/elasticsearch/)中的数据；

`_size`：这个字段存储着_source字段中信息的大小，单位是byte；不过这需要我们安装mapper-size插件。

#### 索引元数据

**Indexing meta-fields**

`_all`：这个字段索引了所有其他字段的值；

`_field_names`：存储着文档中所有值为非空的字段信息，这在快速查找/过滤值存在或者值为空的情况下非常有用；

`_timestamp`：存储着当前文档的时间戳信息，可以由程序指定，也可以由ElasticSearch自动生成，其值会影响文档的删除（如果启用了TTL机制）；

`_ttl`：标识着当前文档存储的时长，超过了这个时长文档将会被标识为delete，之后会被ElasticSearch删除。

#### 路由元数据

**Routing meta-fields**

`_parent`：用于创建两个映射的父子之间的关系；

`_routing`：自定义路由值，可以路由某个文档到具体的分片(shard)。

#### 其他元数据

`_meta`：特定于应用程序的元数据。

### 映射分类

#### **动态映射**

**就是自动创建出来的映射。es 根据存入的文档，自动分析出来文档中字段的类型以及存储方式，这种就是动态映射。**

**有的时候，如果希望新增字段时，能够抛出异常来提醒开发者，这个可以通过 mappings 中 dynamic 属性来配置。**

**dynamic 属性有三种取值：**

* **true，默认即此。自动添加新字段。**
* **false，忽略新字段。**
* **strict，严格模式，发现新字段会抛出异常。**

**映射会根据某个字段插入的值自动判断其字段类型，**

#### 静态映射

**像mysql一样在建表的时候对各个字段的属性进行设置，手动创建映射并指定各个字段类型。如果新增的文档的某个字段和指定的类型不一致，新增就不会成功会报错。**

### 映射的操作

**查看所有映射**

```
GET _mapping
```

**查看某个索引下的映射：**

```
GET index_name/_mapping
```

**创建某个索引下的映射**

```
POST packet_4/_mapping
{
  "properties": {
    "src_ip":{
      "type":"ip"
    }
  }
}
```

**修改索引，可以添加新字段，不能修改已有字段，如下增加一个email字段到映射里**

```
PUT /packet_2/_mapping
{
  "properties": {
    "email": {
      "type": "keyword"
    }
  }
}
```

## 聚合

**这是对ES的数据进行的统计分析的功能，ES的另一个重要功能是搜索。**

**聚合有点像mongodb的aggregate。**

### 分类

#### **指标**

**指标metric聚合**

#### **桶**

**bucketing桶聚合**

**即分桶操作，类似于关系型数据库的group by**

#### **矩阵**

**矩阵聚合**（matrix）

#### **管道**

** 管道聚合（pipleline）**

### 语法

```
"aggregations" : {
    "<aggregation_name>" : { <!--聚合的名字 -->
        "<aggregation_type>" : { <!--聚合的类型 -->
            <aggregation_body> <!--聚合体：对哪些字段进行聚合 -->
        }
        [,"meta" : {  [<meta_data_body>] } ]? <!--元 -->
        [,"aggregations" : { [<sub_aggregation>]+ } ]? <!--在聚合里面在定义子聚合 -->
    }
    [,"<aggregation_name_2>" : { ... } ]*<!--聚合的名字 -->
}
```

**aggregations 也可简写为 aggs**

**比如：**

```
POST /packet_4/_search?
{
  "size": 0, 
  "aggs": {
    "masssbalance": {
      "max": {
        "field": "dst_port"
      }
    }
  }
}
```

**在/packet_4/_search?追加size=0可以让返回结果中的hits字段长度为0，否则的话会返回所有命中的文档影响查看。**

**下面是一些简单的例子：**

**统计数量**

```
POST /bank/_doc/_count
{
  "query": {
    "match": {
      "age" : 24
    }
  }
}
```

**Value count 统计某字段有值的文档数**

**cardinality 值去重计数**

**stats 统计 count max min avg sum 5个值**

**Extended stats，在stats基础上统计平方和、方差、标准差、平均值加/减两个标准差的区间**

**Percentiles 占比百分位对应的值统计**

**Percentiles rank 统计值小于等于指定值的文档占比**

**PS:我写的一些：**

```
POST /packet_4/_search?
{
  "size": 0, 
  "aggs": {
    "masssbalance": {
      "max": {
        "field": "dst_port"
      }
    }
  }
}

POST /packet_4/_search?
{
  "size": 3, 
  "query": {
    "match": {
      "dst_port": 9200
    }
  },
  "sort": [
    {
      "src_port": {
        "order": "desc"
      }
    }
  ],
  "aggs": {
    "max_balance": {
      "max": {
        "field": "src_port"
      }
    }
  }
}

POST /packet_4/_search?size=0
{
  "aggs": {
    "avg_port": {
      "avg": { "field" : "src_port" } 
    },
     "avg_port+10": {
       
      "avg": {  
        "field": "src_port",
        "script": {
            "source": "_value + 10"
        }} 
    }
  }
}

POST /packet_4/_search?size=0
{
  "aggs": {
    "sum_balance": {
      "sum": {
        "field": "src_port",
        "script": {
            "source": "_value * 2.03"
        }
      }
    }
  }
}
```

**详细的语法可以参考第三个参考文档**

## ES字段类型

**字符串类型**： text（字段要被全文检索的话，可以使用该类型，用了 text 之后，字段内容会被分析，在生成倒排索引之前，字符串会被分词器分成一个个词项。text 类型的字段不用于排序，很少用于聚合。这种字符串也被称为 analyzed 字段）keyword（适用于结构化的字段，用作过滤、排序、聚合等。这种字符串也称之为 not-analyzed 字段。）

**数字类型**：long，integer，short，byte，double，float，half_float，scaled_float

**日期类型**：由于 JSON 中没有日期类型，所以 es 中的日期类型形式就比较多样，es 内部存储的是毫秒计时的长整型数。

**布尔类型**boolean：ES解析字段是根据json格式的文档解析的，对于布尔值，可以解析json中的true,false,"true","false"

**二进制类型**binary

**范围类型**：integer_range，float_range，long_range，double_range，date_range，ip_range

**复合类型**：数组，对象object，嵌套类型nested

**地理类型**：geo_point，geo_shape

**特殊类型： IP，token_count**

### IP类型的使用

[原文](https://blog.csdn.net/ubuntutouch/article/details/108517735)

**在使用 Elasticsearch 搜索 IP 地址时，我们可以把数据类型定义为 IP  数据类型。这样我们可以针对 IP 地址进行搜索。这种 IP 地址可以是 IPv4 或者是 IPv6 的形式。**

**现在假设我们导入一个如下的数据到 Elasticsearch 中：**

```
PUT my-index/_doc/1
{
  "ip_addr": "192.168.1.1"
}
```

**在没有定义数据类型的情况下， Elasticsearch 会把上面的字段 ip_add 映射到一个 text  及 keyword 的类型的数据上：**

```
GET my-index/_mapping
```

**上面命令显示的结果为：**

```
{
  "my-index" : {
    "mappings" : {
      "properties" : {
        "ip_addr" : {
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

**假如我们想对上面的数据进行如下的搜索：**

```
GET my-index/_search
{
  "query": {
    "term": {
      "ip_addr": "192.168.0.0/16"
    }
  }
}
```

**针对上面的搜索，我稍微做一下解释：对于上面的 IPv4 的 IP 地址含有4个 bytes，而每个 byte 含有8个 digits。****在上面的 /16 即表示前面的 16 位的 digits，也即 192.168。我们可以这么说任何一个 IP 地址位于 192.168.0.0 至 192.168.255.255 都在这个范围内。**

**上面的搜索的结果是：**

```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```

**也就是找不到任何的结果。这是什么原因呢？**

**就其原因，是因为我们没有正确地把 IP 的数据类型定义为 IP 数据类型。我们重新来定义这个索引的 mapping：**

```
DELETE my-index

PUT my-index
{
  "mappings": {
    "properties": {
      "ip_addr": {
        "type": "ip"
      }
    }
  }
}

PUT my-index/_doc/1
{
  "ip_addr": "192.168.1.1"
}
```

**我们按照上面的步骤来重新建立索引，并导入文档。我们在按照如下的方法来进行搜索：**

```
GET my-index/_search
{
  "query": {
    "term": {
      "ip_addr": "192.168.0.0/16"
    }
  }
}
```

**上面的命令显示的结果为：**

```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "my-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "ip_addr" : "192.168.1.1"
        }
      }
    ]
  }
}

```

**这次，显然我们搜索到我们需要的文档了，这是因为 192.168.1.1 是属于 IP 地址范围 192.168.0.0/16 的。我们可以通过这样的方法搜索属于一个 IP 范围的日志文件供我们查询。**

## 一些补充的命令

**token_count用于统计字符串分词后的词项个数。**

```
PUT blog
{
  "mappings": {
    "properties": {
      "title":{
        "type": "text",
        "fields": {
          "length":{
            "type":"token_count",
            "analyzer":"standard"
          }
        }
      }
    }
  }
}
```

**相当于新增了 title.length 字段用来统计分词后词项的个数。**

**添加文档：**

```
PUT blog/_doc/1
{
  "title":"zhang san"
}
```

**可以通过 token_count 去查询：**

```
GET blog/_search
{
  "query": {
    "term": {
      "title.length": 2
    }
  }
}
```

**创建索引**

```
PUT index_name
```

**更新_id为1的文档：**

```
PUT user/_doc/1
{
  "name": "John",
  "loginCount": 4
}
```

**新建文档：**

```
POST test_index2/_doc
{
  "user":{
      "first":"Zhang121",
      "last":"san1121"
    }
}
```

**查询索引下的文档**

```
GET index_name/_search
```

## 模糊查询

[原文](https://blog.csdn.net/weixin_43859729/article/details/108134329#:~:text=%E5%9C%A8Elasticsearch%E4%B8%AD%EF%BC%8C%E6%88%91%E4%BB%AC%E5%8F%AF%E4%BB%A5,%E8%AF%8D%E7%9A%84%E9%95%BF%E5%BA%A6%E5%AE%9A%E4%B9%89%E8%B7%9D%E7%A6%BB%E3%80%82)

**我们使用关系型数据库时，模糊查询使用的就是like，加上通配符**

**通配符**	**    说明**
**%**	**             包含0个或多个字符的任意字符**
**_（下划线）任意1个字符**
**那ElasticSearch中模糊查询是什么呢，我们知道term是精确查询，有的地方说match是模糊，有的地方说wildcard是模糊，甚至还有fuzzy等，字面意思就是‘模糊’的语句，他们有什么区别呢**

**ElasticSearch中的模糊查询**
**举个例子，我们有个人物名单索引listofhistoricalfigures**
**里面name字段内容如下**

**张三**
**张三丰**
**张飞**
**三德子**
**张二丰**
**孙权**
**马三丰**
**结构是下面这样，text支持分词查询，keyword支持精确查询**
**详情可参考这一篇 ElasticSearch 使用term时.keyword加不加的区别**

```
 "name": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword"
              }
            }
         }
```

**match 分词匹配检索**
**match**
**英 [mætʃ] 美 [mætʃ]**
**n. 火柴;比赛;竞赛;敌手;旗鼓相当的人**
**v.般配;相配;相同;相似;相一致;找相称(或相关)的人(或物);配对**

**match字面意思是 相似;相一致;找相称(或相关)的人(或物);配对**

```
GET listofhistoricalfigures/_search 
'{
    "query": {
        "match": {
            "name": "张三"
        }
    }
}
```

**我们使用match和默认分词器，会把张三进行分词，分成张、三、张三进行检索**
**会匹配到的结果有**

**张三**
**张三丰**
**张飞**
**三德子**
**张二丰**
**马三丰**
**wildcard 通配符检索**
**wildcard**
**美 [ˈwaɪldˌkɑrd]**
**n.未知数;未知因素;(给予没有正常参赛资格的选手准其参加比赛的)“外卡”;“外卡”选手;**
**(用于代替任何字符或字符串的)通配符**

**wildcard字面意思是 通配符**

```
GET listofhistoricalfigures/_search 
'{
    "query": {
        "wildcard": {
            "name.keyword": "张三*"
        }
    }
}
```

**表示匹配0到多个任意字符**
**加.keyword是要匹配完整的词**
**会匹配到的结果有**

**张三**
**张三丰**
**1**
**2**
**fuzzy 模糊/纠错检索**
**fuzzy**
**英 [ˈfʌzi] 美 [ˈfʌzi]**
**adj. 覆有绒毛的;毛茸茸的;紧鬈的;拳曲的;(形状或声音)模糊不清的**

**fuzzy字面意思是 模糊**

```
GET listofhistoricalfigures/_search 
'{
    "query": {
        "fuzzy": {
            "name.keyword": "张三"
        }
    }
}
```

**使用fuzzy就行百度一样，你输入个“邓子棋”，也能把“邓紫棋”查出来，有一定的纠错能力**
**加.keyword是要匹配完整的词**
**会匹配到的结果有**

**张三**
**张三丰**
**张飞**
**张二丰**
**马三丰**
**结论**
**1.match 分词匹配检索，可以对查询条件分词，查到更多匹配的内容，结合不同的分词器，可以得到不同的效果**

**2.wildcard 通配符检索功能就像传统的SQL like一样，如果数据在es，你又想得到传统的“模糊查询”结构时，用wildcard**

**3.fuzzy 纠错检索，让输入条件有容错**

## ES查询语法

**这部分指的是ES的“开发工具”那块的语法。**

```
GET /cars/transactions/_search
{
    "query" : {
        "match" : {
            "make" : "ford"
        }
    },
    "aggs" : {
        "colors" : {
            "terms" : {
              "field" : "color"
            }
        }
    }
}
```

**可以在query里指定查询条件**

## kibana的时区设置

![WX20220712-145321@2x.png](http://tva1.sinaimg.cn/large/006gLprLgy1h445dz1qzuj325w11eh66.jpg)

![WX20220712-145331@2x.png](http://tva1.sinaimg.cn/large/006gLprLgy1h445erz2ecj31f408ggo7.jpg)

**之前ES没有设置用户登录，访问**[http://127.0.0.1:9200](http://127.0.0.1:9200)就会直接返回

```
{
  "name" : "40414b8f8139",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "e9pDFxOIQ2yhNvwJB9153A",
  "version" : {
    "number" : "7.6.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "7f634e9f44834fbc12724506cc1da681b0c3b1e3",
    "build_date" : "2020-02-06T00:09:00.449973Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

**不指定用户名和密码也可以直接往ES写数据，不太安全。可以在需要的时候设置用户名和密码。**

**参考文档**

[ElasticSearch 动态映射和静态映射，以及四种字段类型](https://segmentfault.com/a/1190000038323706)

[Elasticsearch聚合分析简介](https://blog.csdn.net/qq_36918149/article/details/104456820#:~:text=Elasticsearch%20%E6%9C%89%E4%B8%80%E4%B8%AA%E5%8A%9F%E8%83%BD%E5%8F%AB,BY%60%20%E7%B1%BB%E4%BC%BC%E4%BD%86%E6%9B%B4%E5%BC%BA%E5%A4%A7%E3%80%82&text=%E8%81%9A%E5%90%88%E6%A1%86%E6%9E%B6%E6%9C%89%E5%8A%A9%E4%BA%8E,%E5%BC%BA%E5%A4%A7%E7%9A%84%E8%81%9A%E5%90%88%E5%88%86%E6%9E%90%E8%83%BD%E5%8A%9B%E3%80%82)

[elasticsearch系列六：聚合分析（聚合分析简介、指标聚合、桶聚合） ](https://www.cnblogs.com/leeSmall/p/9215909.html)
