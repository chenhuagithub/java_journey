 

## 一、安装

### 普通安装

#### 1.1 安装jdk

#### 1.2 下载elasticsearch安装包

```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.3-linux-x86_64.tar.gz
```

#### 1.3 解压elasticsearch压缩包

```
tar -zxvf tar zxvf elasticsearch-7.9.3-linux-x86_64.tar.gz 
```

### docker安装

#### 1.1 拉取镜像

```
docker pull elasticsearch:7.4.2
```

#### 1.2 启动

```
docker  run --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms64m -Xmx128m" -d elasticsearch:7.4.2
```



##  二、目录说明

[![BHyZtg.png](https://s1.ax1x.com/2020/11/09/BHyZtg.png)](https://imgchr.com/i/BHyZtg)

| 目录      | 作用          | 说明                      |
| ------- | ----------- | ----------------------- |
| bin     | 可执行脚本目录     | elasticsearch启动脚本文件     |
| config  | 用来存放配置文件的目录 | elasticsearch.yml核心配置文件 |
| logs    | 用来存放日志的目录   | 日志                      |
| plugins | 用来扩展ES的插件目录 |                         |

#### 2.1 启动es

> 因为es默认是禁止使用root用户启动，所以我们需要创建一个普通用户

###### 2.1.1 创建用户组

```
groupadd es
```

###### 2.1.2 创建普通用户

```
useradd chenhua -g es
```

###### 2.1.3 修改用户密码:

```
passwd chenhua
```

###### 2.1.4 进入bin目录

```
cd bin
```

[![BHcbXn.png](https://s1.ax1x.com/2020/11/09/BHcbXn.png)](https://imgchr.com/i/BHcbXn)

###### 2.1.5 启动es

```
./elasticsearch
```

###### 2.1.6 测试es是否启动成功

```
curl http://localhost:9200
```

```
{
  "name" : "chenhuacentos",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "CoN_z7xTTumYzXLT8hIQLA",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```





#### 2.2 配置es远程连接

###### 2.2.1 修改配置文件

> 进入config目录， 修改elasticsearch.yml

```
network.host: 0.0.0.0
```

此时启动es会出现以下三个错误

```
ERROR: [3] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[3]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```

> 解决第一个错误

1.修改 /etc/security/limits.conf文件（vim /etc/security/limits.conf）,在文件末尾加入如下内容

```
*        soft        nofile        65536
*        hard        nofile        65536
*        soft        nproc         4096
*        hard        nproc         4096
```

2.退出重新登陆检测配置是否生效（需要root权限）

```
[root@chenhuacentos ~]# ulimit -Hn
65536
[root@chenhuacentos ~]# ulimit -Sn
65536
[root@chenhuacentos ~]# ulimit -Hu
4096
[root@chenhuacentos ~]# ulimit -Su
4096
```

> 解决第二个错误

1.vim /etc/security/limits.d/20-nproc.conf 在文件中修改如下配置

```
chenhua    soft    nproc     4096
root       soft    nproc     unlimited
```

2.vim /etc/sysctl.config 添加如下命令

```
vm.max_map_count=655360
```

3.保存退出后执行如下命令

```
sysctl -p
```

> 解决第三个错误

修改elasticsearch.yml，取消注释保留一个节点

```
cluster.initial_master_nodes: ["node-1"]
```

## 三、ES中的概念

#### 3.1 接近实时(Near Real Time  简称NRT)

> Elasticsearch是一个接近实时的搜索平台。这意味着，从索引一个文档到这个文档能够被搜索到有一个轻微的延迟通讯(通常是1秒内)

#### 3.2 索引(index)

> 一个索引是一个拥有几分相似特征的文档集合。比如说，你可以有一个客户数据的索引，另一个产品目录的索引，还有一个订单数据的索引。一个索引由一个名字来标识(必须全部都是小写字母的),并且当我们要对这个索引中的文档进行索引、搜索、更新和删除的时候，都要使用到这个名字。索引类似于关系型数据库中Database的概念。在一个集群中，如果你想，可以定义任意多个索引

#### 3.3 类型(type)

> 一个类型是你的索引的一个逻辑上的分类/分区，其语义完全由你来定。在一个索引中，你可以定义一种或者多种类型。通常，会为具有一组共同字段的文档定义一个类型。比如，我们假设你运营一个博客平台并且将你所有的数据存储到一个索引中。这个索引中，你可以为用户数据定义一个类型，为博客数据定义另一个类型，当然，也可以为评论数据定义另一个类型。类型类似于关系型数据库中的Table概念
>
> NOTE：在5.x版本以前可以在一个索引中定义多个类型，6.x之后版本也可以使用，但是不推荐，在8.x版本中彻底移除一个索引中创建多个类型。

#### 3.4 映射(Mapping)

> Mapping是ES中的一个很重要的内容，它类似于传统关系型数据库中table的schema，用于定义一个索引(index)中的类型(type)的数据的结构。在ES中，我们可以手动创建type(相当于table)和Mapping(相当于schema),也可以采用默认创建方式。在默认配置下，ES可以根据插入的数据自动地创建type及其mapping。mapping中主要包括字段名、字段数据类型和字段索引类型

#### 3.5 文档(document)

> 一个文档是一个可被索引地基础信息单元，类似于表中地一条记录。比如，你可以拥有一个员工地文档，也可以拥有某个商品的一个文档。文档采用了轻量级的数据交换格式JSON来表示

## 四、安装Kibana

#### 4.1下载镜像

```
docker pull kibana:7.9.3
```

#### 4.2 配置文件

```
mkdir -p /data/elk7/kibana/config/
vim /data/elk7/kibana/config/kibana.yml
```

内容如下：

```
#
# ** THIS IS AN AUTO-GENERATED FILE **
#

# Default Kibana configuration for docker target
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://192.168.31.190:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

**注意：请根据实际情况，修改elasticsearch地址。**

#### 4.3 启动

```
docker run -d \
  --name=kibana \
  --restart=always \
  -p 5601:5601 \
  -v /data/elk7/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml \
  kibana:7.9.3
```

#### 4.4 访问页面

```
http://192.168.31.190:5601/
```

## 五、Kibana基本操作

#### 5.1 索引(index)的基本操作

```kibana
PUT /dangdang/    创建索引
DELETE /dangdang  删除索引
DELETE /*         删除所有索引
GET /_cat/indices?v  查看索引信息
```

#### 5.2 类型(type)的基本操作

> 创建索引和类型

```http
PUT /ems
{
  "mappings":{
    "properties":{
      "id":{"type": "keyword"},
      "name":{"type": "keyword"},
      "age":{"type": "integer"},
      "bir":{"type": "date"}
    }
  }
}
注意： 这种方式创建类型要求索引不能存在
```

> Mapping Type: text, keyword, date, integer, long, double, boolean or ip

#### 5.3 文档(document)的基本操作

###### 5.3.1 添加文档

```http
PUT /dangdang/_doc/1  #/索引/类型/_id
{
  "id": "21",
  "name": "小黑",
  "price": 23.23,
  "counts": 15
}
```

###### 5.3.2 查询文档

```http
GET /dangdang/_doc/1
返回结果：
{
  "_index" : "dangdang",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "id" : "21",
    "name" : "小黑",
    "price" : 23.23,
    "counts" : 15
  }
}
```

###### 5.3.3 更新文档

> 使用PUT更新文档，相当于删除原来的文档，在新建新的文档

```http
PUT /dangdang/_doc/1
{
  "id": "21",
  "name": "小黑",
  "price": 11.11,
  "counts": 15
}
```

> 使用_update更新文档

```http
POST /dangdang/_update/1
{
  "doc":{
    "name": "张三",
    "gender": "男"
  }
}
```

> 使用script更新,只能更新ingeger和double类型数据

```http
POST /dangdang/_doc/1/_update
{
  "script":"ctx._source.price += 5"
}
```

###### 5.3.4 删除文档

```http
DELETE /dangdang/_doc/1
```



## 六、ES高级检索

#### 6.1 检索方式

>ES官方提供了两种检索方式：一种是通过URL参数进行搜索，另一种是通过DSL进行搜索。官方更推荐使用第二种方式，因为第二种方式是基于传递JSON作为请求体格式与ES进行交互，这种方式更强大，更简洁。

#### 6.2 URL

###### 6.2.1 查询所有

```http
GET /ems/_search?q=*
```

###### 6.2.2 查询并排序

```http
GET /ems/_search?q=*&sort=age:desc
```

###### 6.2.3 查询并限制展示文档的数目

```http
GET /ems/_search?q=*&sort=age:desc&size=2
```

###### 6.2.4 分页查询

```http
GET /ems/_search?q=*&sort=age:desc&size=2&from=2
```

#### 6.3 DSL

  ###### 6.3.1 查询所有(match_all)

```http
GET /ems/_search
{
  "query": {"match_all": {}}
}
```

###### 6.3.2 查询结果中返回指定条数(size)

```http
GET /ems/_search
{
  "query": {"match_all": {}},
  "size": 3
}
```

###### 6.3.3 分页查询(from)

```http
GET /ems/_search
{
  "query": {"match_all": {}},
  "size": 3,
  "from": 1
}
```

###### 6.3.4 排序(sort)

```http
GET /ems/_search
{
  "query": {"match_all": {}},
  "sort": [
    {
      "age": {
        "order": "desc"
      }
    }
  ]
}
```

###### 6.3.5 查询结果中返回指定字段(_source)

> _source关键字：是一个数组，在数组中用来指定展示那些字段

```http
GET /ems/_search
{
  "query": {"match_all": {}},
  "_source": ["name", "age"]
}
```

###### 6.3.6 精确查询(term)和一般查询(match)

> term关键词：用来使用关键词查询

```http
GET /ems/_search
{
  "query": {
    "term": {
      "content": {
        "value": "成龙"
      }
    }
  }
}
```

> match关键词：在查询前会对查询内容分词

```http
GET /ems/_search
{
  "query": {
    "match": {
      "content": "成龙赵露思"
    }
  }
}
```

> note1：通过使用term查询得知ES中默认使用分词为标准分词器，标准分词器对于英文单词分词，对于中文单字分词。
>
> note2：通过使用term查询得知，在ES的Mapping Type中keyword，date，integer，long，double，boolean or ip这些类型不分词，只有text类型分词。

###### 6.3.7 范围查询(range)

> range关键字：用来指定查询指定范围内的文档

```http
GET /ems/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 10,
        "lte": 30
      }
    }
  }
}
```

###### 6.3.8 前缀查询(prefix)

> prefix关键字：用来检索含有指定前缀词的相关文档

```http
GET /ems/_search
{
  "query": {
    "prefix": {
      "content": {
        "value": "露"
      }
    }
  }
}
```

###### 6.3.9 通配符查询(wildcard)

> wildcard关键字：通配符查询，`？`用来匹配一个任意字符，`*`用来匹配多个任意字符,不能用于text类型的数据

```http
GET /ems/_search
{
  "query": {
    "wildcard": {
      "name": {
        "value": "赵*思"
      }
    }
  }
}
```

###### 6.3.10 多id查询(ids)

> ids关键字：值为数组类型，用来根据一组id获取多个对应的文档

```http
GET /ems/_search
{
  "query": {
    "ids": {
      "values": ["m-xx23UBYLiWt0j-u8Wg", "nexx23UBYLiWt0j-u8Wg"]
    }
  }
}

```

###### 6.3.11 模糊查询(fuzzy)

> fuzzy关键字：用来模糊查询含有指定关键字的文档。
>
> 注意：允许出现的错误必须在0-2之间。

```http
GET /ems/_search
{
  "query": {
    "fuzzy": {
      "content": "spr00ng"
    }
  }
}

```

> 注意：最大编辑距离为 0 1 2
>
> 如果关键词为2个长度，必须完全匹配
>
> 如果关键字长度为3到5之间，允许一个失败
>
> 如果关键字长度>5,最多允许两个错误

###### 6.3.12 布尔查询(bool)

> bool关键字：用来组合多个条件实现复杂查询
>
> ​	must: 相当于&&，同时成立
>
> ​	should: 相当于||，成立一个就行
>
> ​	must not: 相当于！，不能满足任何一个

```http
GET /ems/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "wildcard": {
            "name": {
              "value": "赵*思"
            }
          }
        }
      ],
      "must_not": [
        {
          "ids": {
            "values": ["sOxR3HUBYLiWt0j-1cf7"]
          }
        }
      ]
    }
  }
}
```

###### 6.3.13 多字段查询(multi_match)

> 注意：使用这种方式进行查询时，为了更好获取搜索结果，在查询过程中先将查询条件根据当前的分词器分词之后进行查询

```http
GET /ems/_search
{
  "query": {
    "multi_match": {
      "query": "露思",
      "fields": ["content"]
    }
  }
}
```

###### 6.3.14 多字段分词查询(query_string)

> 注意：使用这种方式进行查询时，为了更好获取搜索结果，在查询过程中先将查询条件根据当前的分词器分词之后进行查询。

```http
GET /ems/_search
{
  "query": {
    "query_string": {
      "fields": ["name", "cont ent"],
      "query": "赵露",
      "analyzer": "standard"
    }
  }
}

注意：这种方式可以指定分词器
```

###### 6.3.15 高亮匹配

> 注意：只有允许分词的字段才会进行高亮匹配

```http
GET /ems/_search
{
  "query": {
    "term": {
      "content": {
        "value": "成"
      }
    }
  },
  "highlight": {
    "fields": {"*": {}},
    "pre_tags": ["<span style = 'color:red'>"], # 高亮前缀
    "post_tags": ["</span>"],  # 高亮后缀
    "require_field_match": "false"
  }
}
```

#### 6.3 过滤查询

> 其实准确来说，ES中的查询操作分为2种：查询(query)和过滤(filter)。查询即是之前提到的query查询，它(查询)就会计算每个返回文档的得分，然后根据得分排序。而过滤(filter)只会筛选出符合的文档，并不计算得分，且它可以缓存文档。所以，单从性能考虑，过滤比查询更快。
>
> ​	换句话说，过滤适合在大范围帅选数据，而查询则适合精确匹配数据。一般应用时，应先使用过滤操作过滤数据，然后使用查询匹配数据。

[![Dtl5pn.png](https://s3.ax1x.com/2020/11/24/Dtl5pn.png)](https://imgchr.com/i/Dtl5pn)

###### 6.3.1 过滤语法

`range filter`

```http
GET /ems/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "成龙赵露思"
          }
        }
      ],
      "filter": [
        {
          "range": {
            "age": {
              "gte": 20,
              "lte": 30
            }
          }
        }
      ]
    }
  }
}
```

> note: 在执行filter和query的顺序是，先执行filter再执行query
>
> note：elasticsearch会自动缓存经常使用的过滤器，以加快性能

`term filter & terms filter`

```http
GET /ems/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "成龙赵露思"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "name": "赵"
          }
        }
      ]
    }
  }
}
```

```http
GET /ems/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "成龙赵露思"
          }
        }
      ],
      "filter": [
        {
          "terms": {
            "content": [
              "四川",
              "阿姨"
            ]
          }
        }
      ]
    }
  }
}
```

`exists filter`

> 过滤存在指定字段，获取字段不为空的索引记录使用

```http
GET /ems/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "成龙赵露思"
          }
        }
      ],
      "filter": [
        {
          "exists": {
            "field": "address"
          }
        }
      ]
    }
  }
}
```

## 七、分词器

> ES提供了两种分词器
>
> 1. standard analyzer 
> 2. simple analyzer

|        分词器        |  中文  |  英文  |
| :---------------: | :--: | :--: |
| standard analyzer | 单字分词 | 单词分词 |
|  simple analyzer  | 不分词  | 单词分词 |



#### 7.1 standard analyzer

> 测试分词效果

```http
GET _analyze
{
  "analyzer": "standard",
  "text": "redis 非常好用111!"
}

```

> 结果

```
{
  "tokens" : [
    {
      "token" : "redis",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "非",
      "start_offset" : 5,
      "end_offset" : 6,
      "type" : "<IDEOGRAPHIC>",
      "position" : 1
    },
    {
      "token" : "常",
      "start_offset" : 6,
      "end_offset" : 7,
      "type" : "<IDEOGRAPHIC>",
      "position" : 2
    },
    {
      "token" : "好",
      "start_offset" : 7,
      "end_offset" : 8,
      "type" : "<IDEOGRAPHIC>",
      "position" : 3
    },
    {
      "token" : "用",
      "start_offset" : 8,
      "end_offset" : 9,
      "type" : "<IDEOGRAPHIC>",
      "position" : 4
    },
    {
      "token" : "111",
      "start_offset" : 9,
      "end_offset" : 12,
      "type" : "<NUM>",
      "position" : 5
    }
  ]
}
```

#### 7.2 simple analyzer

> 测试分词效果

```http
GET _analyze
{
  "analyzer": "simple",
  "text": "redis 非常好用111!"
}

```

> 结果

```http
{
  "tokens" : [
    {
      "token" : "redis",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "非常好用",
      "start_offset" : 6,
      "end_offset" : 10,
      "type" : "word",
      "position" : 1
    }
  ]
}

```

#### 7.3 ik analyzer

###### 7.3.1 安装

> 进入bin目录下，执行以下命令

```http
./elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.9.3/elasticsearch-analysis-ik-7.9.3.zip
```

###### 7.3.2 ik_max_word

> 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合，适合 Term Query

```http
GET _analyze
{
  "analyzer": "ik_max_word",
  "text": "中华人民共和国国歌"
}
```

###### ik_smart

> 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”，适合 Phrase 查询

```http
GET _analyze
{
  "analyzer": "ik_smart",
  "text": "中华人民共和国国歌"
}
```

###### 使用



## 8、SpringBoot操作ES

[![DNm6bQ.png](https://s3.ax1x.com/2020/11/24/DNm6bQ.png)](https://imgchr.com/i/DNm6bQ)



#### 8.1 引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

#### 8.2 连接ES

```java
@Configuration
public class RestClientConfig extends AbstractElasticsearchConfiguration {
    @Override
    @Bean
    public RestHighLevelClient elasticsearchClient() {
        final ClientConfiguration clientConfiguration = ClientConfiguration.builder()
                .connectedTo("192.168.56.110:9200")
                .build();
        return RestClients.create(clientConfiguration).rest();
    }
}
```

#### 8.3 RestHighLevelClient方式

###### 8.3.1 删除文档

> ```java
> /**
> * @index  索引
> * @type   类型
> * @id     id
> */
> public DeleteRequest(String index, String type, String id) {
>     this.versionType = VersionType.INTERNAL;
>     this.ifSeqNo = -2L;
>     this.ifPrimaryTerm = 0L;
>     this.index = index;
>     this.type = type;
>     this.id = id;
> }
> ```

```java
@Autowired
private RestHighLevelClient restHighLevelClient;

@Test
void contextLoads() throws IOException {
    DeleteRequest deleteRequest = new DeleteRequest("ems", "_doc", "j1sB9XUBaLBWCWet6d2y");
    DeleteResponse deleteResponse = restHighLevelClient.delete(deleteRequest, RequestOptions.DEFAULT);
    System.out.println(deleteResponse.status());
}
```

###### 8.3.2 添加文档

> ```java
> /**
> * @index  索引
> * @type   类型
> * @id     id
> */
> public IndexRequest(String index, String type, String id) {
>     this.opType = OpType.INDEX;
>     this.version = -3L;
>     this.versionType = VersionType.INTERNAL;
>     this.autoGeneratedTimestamp = -1L;
>     this.isRetry = false;
>     this.ifSeqNo = -2L;
>     this.ifPrimaryTerm = 0L;
>     this.index = index;
>     this.type = type;
>     this.id = id;
> }
> ```

```java
@Autowired
private RestHighLevelClient restHighLevelClient;

@Test
void testAddIndex () throws IOException {
    IndexRequest indexRequest = new IndexRequest("ems", "_doc", "12");
    IndexRequest source = indexRequest.source("{\"name:\":\"关羽\",\"age\":44}", XContentType.JSON);
    IndexResponse indexResponse = restHighLevelClient.index(indexRequest, RequestOptions.DEFAULT);
    System.out.println(indexResponse.status());
}
```

###### 8.3.3 查询文档

```java
@Autowired
private RestHighLevelClient restHighLevelClient;

@Test
void testSearch () throws IOException {
    SearchRequest searchRequest = new SearchRequest("ems");
    // 构建搜索对象
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder
        .query(QueryBuilders.matchAllQuery()) // 执行查询条件
        .from(0) // 起始条数
        .size(3) // 每页展示记录
        .postFilter(QueryBuilders.matchAllQuery()) // 过滤条件
        .sort("age", SortOrder.DESC) // 排序
        .highlighter(new HighlightBuilder().field("*").requireFieldMatch(false)); // 高亮匹配
    // 创建搜索请求
    searchRequest.source(searchSourceBuilder);
    SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println("符合条件的文档总数：" + searchResponse.getHits().getTotalHits());
    System.out.println("符合条件的文档最大得分：" + searchResponse.getHits().getMaxScore());
    SearchHit[] hits = searchResponse.getHits().getHits();
    for (SearchHit hit : hits) {
        System.out.println(hit.getSourceAsMap());
    }
}
```

###### 8.3.4 更新文档

```java
@Autowired
private RestHighLevelClient restHighLevelClient;

@Test
void testUpdate () throws IOException {
    UpdateRequest updateRequest = new UpdateRequest("ems", "_doc", "UBgq43UBYw_0039FP3KK");
    updateRequest.doc("{\"name\":\"赵露思\", \"age\":122}", XContentType.JSON);
    UpdateResponse updateResponse = restHighLevelClient.update(updateRequest, RequestOptions.DEFAULT);
    System.out.println(updateResponse.status());
}
```

#### 8.3.5 高亮

```java
    @GetMapping("/hightlight")
    public List<Emp> higtlight (String content, Integer from, Integer size) throws IOException, ParseException {
        // 创建搜索请求
        SearchRequest searchRequest = new SearchRequest("ems");
        // 创建搜索对象
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder
                .query(QueryBuilders.matchQuery("content", content))
                .sort("age", SortOrder.DESC)
                .from(from)
                .size(size)
                .highlighter(new HighlightBuilder().field("*").requireFieldMatch(false).preTags("<span style='color:red'>").postTags("</span>")); // 设置高亮
//        searchRequest.types("_doc").source(searchSourceBuilder); // 过期
        searchRequest.source(searchSourceBuilder);
        // 执行搜索
        SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        // 获取结果
        SearchHit[] hits = searchResponse.getHits().getHits();
        List<Emp> list = new ArrayList<>();
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            Emp emp = new Emp();
            // 原始字段
            emp.setId(hit.getId());
            emp.setAge(Integer.parseInt(sourceAsMap.get("age").toString()));
//            emp.setBir(new SimpleDateFormat("yyyy-MM-dd").format(new Date(Long.parseLong("1607788800000"))));
            emp.setContent(sourceAsMap.get("content").toString());
            emp.setName(sourceAsMap.get("name").toString());
            emp.setAddress(sourceAsMap.get("address").toString());

            // 高亮字段
            Map<String, HighlightField> highlightField = hit.getHighlightFields();
            if (highlightField.containsKey("content")) {
                emp.setContent(highlightField.get("content").fragments()[0].toString());
            }
            if (highlightField.containsKey("name")) {
                emp.setName(highlightField.get("name").fragments()[0].toString());
            }
            if (highlightField.containsKey("address")) {
                emp.setAddress(highlightField.get("address").fragments()[0].toString());
            }
            list.add(emp);
        }
        return list;
    }
```



#### 8.4 ElasticsearchCrudRepository方式

###### 8.4.1 定义bean

```java
package com.journey.bootes.elasticsearch.bean;


import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

import java.util.Date;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Document(indexName = "ems")
public class Emp {
    @Id // 用来将对象中id属性与文档中_id一一对应
    private String id;
    @Field(type = FieldType.Text, analyzer = "ik_max_word")
    private String name;
    @Field(type = FieldType.Integer)
    private Integer age;
    @Field(type = FieldType.Date)
    private Date bir;
    @Field(type = FieldType.Text, analyzer = "ik_max_word")
    private String content;
    @Field(type = FieldType.Text, analyzer = "ik_max_word")
    private String address;
}
```

###### 8.4.2 接口实现

```java
package com.journey.bootes.elasticsearch.dao;

import com.journey.bootes.elasticsearch.bean.Emp;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;


// 自定义EmpRespository
public interface EmpRespository extends ElasticsearchRepository<Emp, String> {
}

```

###### 8.4.3 添加文档

```java
@Autowired
private EmpRespository empRespository;

@PostMapping("/add")
public String method (Emp emp) {
    empRespository.save(emp);
    return "成功";
}
```

###### 8.4.4 查询文档

```java
@Autowired
private EmpRespository empRespository;

@GetMapping("/findOne")
public Emp searchOne (String id) {
    Optional<Emp> byId = empRespository.findById(id);
    return byId.get();
}

@GetMapping("findAll")
public List<Emp> findAll () {
    Iterable<Emp> emps = empRespository.findAll();
    List<Emp> list = new ArrayList<>();
    Iterator<Emp> iterator = emps.iterator();
    while (iterator.hasNext()) {
        list.add(iterator.next());
    }
    return list;
}
```

###### 8.4.5 删除文档

```java
@Autowired
private EmpRespository empRespository;

@GetMapping("/delete")
public String delete (String id) {
    empRespository.deleteById(id);
    return "删除成功";
}

@GetMapping("/deleteAll")
public String deleteAll () {
    empRespository.deleteAll();
    return "删除所有成功";
}
```

###### 8.4.6 排序

```java
@GetMapping("/sort")
public List<Emp> findSort () {
    Iterable<Emp> emps = empRespository.findAll(Sort.by(Sort.Order.asc("age")));
    List<Emp> list = new ArrayList<>();
    Iterator<Emp> iterator = emps.iterator();
    while (iterator.hasNext()) {
        list.add(iterator.next());
    }
    return list;
}
```

###### 8.4.7 分页

```java
@GetMapping("/page")
public List<Emp> findPage (int page, int size) {
    Page<Emp> emps = empRespository.search(QueryBuilders.matchAllQuery(), PageRequest.of(page, size));
    List<Emp> list = new ArrayList<>();
    for (Emp emp : emps) {
        list.add(emp);
    }
    return list;
}
```

###### 8.4.8 自定义操作

| Keyword                                  | Sample                                   | Elasticsearch Query String               |
| :--------------------------------------- | :--------------------------------------- | :--------------------------------------- |
| `And`                                    | `findByNameAndPrice`                     | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } }, { "query_string" : { "query" : "?", "fields" : [ "price" ] } } ] } }}` |
| `Or`                                     | `findByNameOrPrice`                      | `{ "query" : { "bool" : { "should" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } }, { "query_string" : { "query" : "?", "fields" : [ "price" ] } } ] } }}` |
| `Is`                                     | `findByName`                             | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } } ] } }}` |
| `Not`                                    | `findByNameNot`                          | `{ "query" : { "bool" : { "must_not" : [ { "query_string" : { "query" : "?", "fields" : [ "name" ] } } ] } }}` |
| `Between`                                | `findByPriceBetween`                     | `{ "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : ?, "include_lower" : true, "include_upper" : true } } } ] } }}` |
| `LessThan`                               | `findByPriceLessThan`                    | `{ "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : null, "to" : ?, "include_lower" : true, "include_upper" : false } } } ] } }}` |
| `LessThanEqual`                          | `findByPriceLessThanEqual`               | `{ "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : null, "to" : ?, "include_lower" : true, "include_upper" : true } } } ] } }}` |
| `GreaterThan`                            | `findByPriceGreaterThan`                 | `{ "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : null, "include_lower" : false, "include_upper" : true } } } ] } }}` |
| `GreaterThanEqual`                       | `findByPriceGreaterThan`                 | `{ "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : null, "include_lower" : true, "include_upper" : true } } } ] } }}` |
| `Before`                                 | `findByPriceBefore`                      | `{ "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : null, "to" : ?, "include_lower" : true, "include_upper" : true } } } ] } }}` |
| `After`                                  | `findByPriceAfter`                       | `{ "query" : { "bool" : { "must" : [ {"range" : {"price" : {"from" : ?, "to" : null, "include_lower" : true, "include_upper" : true } } } ] } }}` |
| `Like`                                   | `findByNameLike`                         | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?*", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }}` |
| `StartingWith`                           | `findByNameStartingWith`                 | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "?*", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }}` |
| `EndingWith`                             | `findByNameEndingWith`                   | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "*?", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }}` |
| `Contains/Containing`                    | `findByNameContaining`                   | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "*?*", "fields" : [ "name" ] }, "analyze_wildcard": true } ] } }}` |
| `In` (when annotated as FieldType.Keyword) | `findByNameIn(Collection<String>names)`  | `{ "query" : { "bool" : { "must" : [ {"bool" : {"must" : [ {"terms" : {"name" : ["?","?"]}} ] } } ] } }}` |
| `In`                                     | `findByNameIn(Collection<String>names)`  | `{ "query": {"bool": {"must": [{"query_string":{"query": "\"?\" \"?\"", "fields": ["name"]}}]}}}` |
| `NotIn` (when annotated as FieldType.Keyword) | `findByNameNotIn(Collection<String>names)` | `{ "query" : { "bool" : { "must" : [ {"bool" : {"must_not" : [ {"terms" : {"name" : ["?","?"]}} ] } } ] } }}` |
| `NotIn`                                  | `findByNameNotIn(Collection<String>names)` | `{"query": {"bool": {"must": [{"query_string": {"query": "NOT(\"?\" \"?\")", "fields": ["name"]}}]}}}` |
| `Near`                                   | `findByStoreNear`                        | `Not Supported Yet !`                    |
| `True`                                   | `findByAvailableTrue`                    | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "true", "fields" : [ "available" ] } } ] } }}` |
| `False`                                  | `findByAvailableFalse`                   | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "false", "fields" : [ "available" ] } } ] } }}` |
| `OrderBy`                                | `findByAvailableTrueOrderByNameDesc`     | `{ "query" : { "bool" : { "must" : [ { "query_string" : { "query" : "true", "fields" : [ "available" ] } } ] } }, "sort":[{"name":{"order":"desc"}}] }` |

```java
public interface EmpRespository extends ElasticsearchRepository<Emp, String> {
    List<Emp> findByName(String name);

    List<Emp> findByAgeBetween (int start, int end);
}
```
