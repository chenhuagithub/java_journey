# canal adapter的使用

这里只介绍adapter的使用，有关环境的搭建请看上一篇环境搭建的教程

> 注意：
>
> canal adapter和es(这里我们研究的是es7)之间的数据同步是通过适配器表映射文件来完成的, 通过在映射文件中写符合官方规定规则的sql语句实现数据的同步，具体canal adapter是如何实现数据的同步就需要各位大佬去看看源码了，我们现阶段只需要知道是，配置了映射文件，canal adapter会自动帮助我们实现数据的同步。最值得注意的是，我们要事先在es7中创建好sql语句对应索引文件，canal adapter是不会帮助我们自动创建索引的，官网并没有明确说到这一点，这也是很多人在学习canal的过程中踩了很多坑的原因，包括我在内，我一开始也不知道要事先建立好对应的索引，所有一直实现不了数据的同步。



温馨提示：项目中在映射文件修改后一定要对整个client-adapter项目进行mvn install，否则无法识别映射文件的更改。



## 映射文件特别说明

下面的文件和说明摘自官网。

application.yml

```yaml
server:
  port: 8081
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null

canal.conf:
  mode: tcp #tcp kafka rocketMQ rabbitMQ
  flatMessage: true
  zookeeperHosts:
  syncBatchSize: 1000
  retries: 0
  timeout:
  accessKey:
  secretKey:
  consumerProperties:
    # canal tcp consumer
    canal.tcp.server.host: 127.0.0.1:11111
    canal.tcp.zookeeper.hosts:
    canal.tcp.batch.size: 500
    canal.tcp.username:
    canal.tcp.password:
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://127.0.0.1:3306/mytest?useUnicode=true
      username: root
      password: 12345678
  canalAdapters:
  - instance: example # canal instance Name or mq topic name
    groups:
    - groupId: g1
      outerAdapters:
      - name: logger
      - name: es7
        hosts: 127.0.0.1:9300 # 127.0.0.1:9200 for rest mode
        properties:
          mode: transport # or rest
          # security.auth: test:123456 #  only used for rest mode
          cluster.name: docker-cluster
```

mytest_user.yml映射文件

```yaml
dataSourceKey: defaultDS        # 源数据源的key, 对应上面配置的srcDataSources中的值
outerAdapterKey: exampleKey     # 对应application.yml中es配置的key 
destination: example            # cannal的instance或者MQ的topic
groupId:                        # 对应MQ模式下的groupId, 只会同步对应groupId的数据
esMapping:
  _index: mytest_user           # es 的索引名称
  _type: _doc                   # es 的type名称, es7下无需配置此项
  _id: _id                      # es 的_id, 如果不配置该项必须配置下面的pk项_id则会由es自动分配
#  pk: id                       # 如果不需要_id, 则需要指定一个属性为主键属性
  # sql映射
  sql: "select a.id as _id, a.name as _name, a.role_id as _role_id, b.role_name as _role_name,
        a.c_time as _c_time, c.labels as _labels from user a
        left join role b on b.id=a.role_id
        left join (select user_id, group_concat(label order by id desc separator ';') as labels from label
        group by user_id) c on c.user_id=a.id"
#  objFields:
#    _labels: array:;           # 数组或者对象属性, array:; 代表以;字段里面是以;分隔的
#    _obj: object               # json对象
  etlCondition: "where a.c_time>='{0}'"     # etl 的条件参数
  commitBatch: 3000                         # 提交批大小
```

sql映射说明:

sql支持多表关联自由组合, 但是有一定的限制:

1. 主表不能为子查询语句
2. 只能使用left outer join即最左表一定要是主表
3. 关联从表如果是子查询不能有多张表
4. 主sql中不能有where查询条件(从表子查询中可以有where条件但是不推荐, 可能会造成数据同步的不一致, 比如修改了where条件中的字段内容)
5. 关联条件只允许主外键的'='操作不能出现其他常量判断比如: on a.role_id=b.id and b.statues=1
6. 关联条件必须要有一个字段出现在主查询语句中比如: on a.role_id=b.id 其中的 a.role_id 或者 b.id 必须出现在主select语句中

Elastic Search的mapping 属性与sql的查询值将一一对应(不支持 select *), 比如: select a.id as _id, a.name, a.email as _email from user, 其中name将映射到es mapping的name field, _email将 映射到mapping的_email field, 这里以别名(如果有别名)作为最终的映射字段. 这里的_id可以填写到配置文件的 _id: _id映射.



### 1. 单表映射

映射文件

```xml
dataSourceKey: defaultDS
destination: example
groupId: g1
esMapping:
  _index: mytest_single
  _id: _id
  sql: "select a.id as _id, a.name, a.role_id, a.c_time from user a"
  etlCondition: "where a.c_time>={}"
  commitBatch: 3000

```

es索引

```json
PUT /mytest_single
{
  "mappings":{
    "properties":{
      "name":{"type": "text"},
      "role_id":{"type": "integer"},
      "c_time":{"type": "date"}
    }
  }
}
```

测试：

我们在mysql中插入一条数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804112825.png)

idea控制台

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804120843.png)

查看es

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804120946.png)



### 2. 多表映射

#### 2.1 一对一/多对一

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804122450.png)

映射文件

```yaml
dataSourceKey: defaultDS
destination: example
groupId: g1
esMapping:
  _index: mytest_more_to_one
  _id: _id
  #  upsert: true
  #  pk: id
  sql: "select a.id as _id, a.name, a.role_id, b.role_name, a.c_time from user a
        left join role b on b.id = a.role_id"
  #  objFields:
  #    _labels: array:;
  etlCondition: "where a.c_time>={}"
  commitBatch: 3000
```

索引

```json
PUT /mytest_more_to_one
{
  "mappings":{
    "properties":{
      "name":{"type": "text"},
      "role_id":{"type": "integer"},
      "role_name":{"type": "text"},
      "c_time":{"type": "date"}
    }
  }
}
```

数据库插入数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804122613.png)

控制台日志

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804122700.png)

查看es

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804122732.png)

#### 2.2 一对多

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804150156.png)

映射文件

```yaml
dataSourceKey: defaultDS
destination: example
groupId: g1
esMapping:
  _index: mytest_one_to_more
  _id: _id
  #  upsert: true
  #  pk: id
  sql: "select a.id as _id, a.name, a.role_id, c.labels, a.c_time from user a
        left join (select user_id, group_concat(label_name order by id desc separator ';') as labels from label
                group by user_id) c on c.user_id=a.id"
  objFields:
    labels: array:; # 以；为标志对数据进行分割
  etlCondition: "where a.c_time>={}"
  commitBatch: 3000

```

es索引

```json
PUT /mytest_one_to_more
{
  "mappings":{
    "properties":{
      "name":{"type": "text"},
      "role_id":{"type": "integer"},
      "c_time":{"type": "date"},
      "labels":{"type": "text"}
    }
  }
}
```

在数据库插入一条数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804152939.png)

控制台日志

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804152854.png)

查看es

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804153056.png)



### 3. 其他类型的sql示例

具体的操作同上面一样，这里只贴相应的sql语句和es索引

- 复合主键

```mysql
sql:"select concat(a.id,'_',b.id) as _id, a.name, b.role_name from user a left join role b on b.id=a.role_id";
```

es索引

```json
PUT /mytest_complex_id
{
  "mappings":{
    "properties":{
      "name":{"type": "text"},
      "role_name":{"type": "text"}
    }
  }
}
```

- 数组字段

```mysql
sql:"select a.id as _id, a.name, a.role_id, c.labels, a.c_time from user a
        left join (select user_id, group_concat(label_name order by id desc separator ';') as labels from label
                group by user_id) c on c.user_id=a.id"
```

配置中使用：

```yaml
objFields:
  labels: array:;
```

es索引

```json
PUT /mytest_one_to_more
{
  "mappings":{
    "properties":{
      "name":{"type": "text"},
      "role_id":{"type": "integer"},
      "c_time":{"type": "date"},
      "labels":{"type": "text"}
    }
  }
}
```

- 对象字段

```yaml
sql:"select a.id as _id, a.name, a.role_id, a.c_time, a.description from user a
"
```

配置中使用：

```yaml
objFields:
  description: object
```

其中a.description字段内容为json字符串

es索引

```json
PUT /mytest_object
{
  "mappings":{
    "properties":{
      "name":{"type": "text"},
      "role_id":{"type": "integer"},
      "c_time":{"type": "date"},
      "description":{
        "properties": {
          "id":{"type": "integer"},
          "username":{"type": "text"},
          "password":{"type": "text"}
        }
      }
    }
  }
}
```





### 4.父子文档

我们上面讨论的所有用法都是局限于单库，有些人可能就有疑问了，我能不能跨库进行关联查询？答案是用传统方法行不通，细心的朋友可以发现，es模块中，每一个映射文件只能配置一个数据源，这也就意味这我们只能操作单库。

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210805101650.png)

那有没有什么方法实现跨库查询呢？答案是肯定的，我们可以利用es的`父子文档`进行实现，`父子文档`中的父文档和子文档是互相独立的，数据可以来自不同的数据库，我们可以利用这一点来实现db的跨库关联查询，刚好canal也支持父子文档的数据同步。



1. 多数据源配置

要实现跨库查询，多数据源配置必不可少，多数据源配置如下：

```yaml
server:
  port: 8081
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null

canal.conf:
  mode: tcp #tcp kafka rocketMQ rabbitMQ
  flatMessage: true
  zookeeperHosts:
  syncBatchSize: 1000
  retries: 0
  timeout:
  accessKey:
  secretKey:
  consumerProperties:
    # canal tcp consumer
    canal.tcp.server.host: 127.0.0.1:11111
    canal.tcp.zookeeper.hosts:
    canal.tcp.batch.size: 500
    canal.tcp.username:
    canal.tcp.password:
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://127.0.0.1:3306/mytest?useUnicode=true
      username: root
      password: 12345678
    mytest2DS:
      url: jdbc:mysql://127.0.0.1:3306/mytest2?useUnicode=true
      username: root
      password: 12345678
    mytest3DS:
      url: jdbc:mysql://127.0.0.1:3306/mytest3?useUnicode=true
      username: root
      password: 12345678
  canalAdapters:
  - instance: example # canal instance Name or mq topic name
    groups:
    - groupId: g1
      outerAdapters:
      - name: logger
      - name: es7
        hosts: 127.0.0.1:9300 # 127.0.0.1:9200 for rest mode
        properties:
          mode: transport # or rest
          cluster.name: docker-cluster
```

2. 数据库关系

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210805103759.png)

3. es7映射文件

customer.yml

```yaml
dataSourceKey: mytest2DS
destination: example
groupId: g1
esMapping:
  _index: customer
  _id: id
  relations:
    customer_order:
      name: customer
  sql: "select t.id, t.name, t.email from customer t"
  commitBatch: 3000
```

biz_order.yml

```yaml
dataSourceKey: mytest3DS
destination: example
groupId: g1
esMapping:
  _index: customer
  _id: _id
  relations:
    customer_order:
      name: order
      parent: customer_id
  sql: "select concat('oid_', t.id) as _id,
        t.customer_id,
        t.id as order_id,
        t.serial_code as order_serial,
        t.c_time as order_time
        from biz_order t"
  skips:
    - customer_id
  etlCondition: "where t.c_time>={}"
  commitBatch: 3000
```

4. es索引

```json
PUT /customer
{
  "mappings":{
    "properties":{
      "id":{"type": "integer"},
      "name":{"type": "text"},
      "email":{"type": "text"},
      "order_id":{"type": "integer"},
      "order_serial":{"type": "text"},
      "order_time":{"type": "date"},
      "customer_order":{
        "type": "join",
        "relations":{
          "customer":"order"
        }
      }
    }
  }
}
```

5. 在数据库插入数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210805103037.png)

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210805103117.png)

6. 查看es数据

```json
GET /customer/_search?q=*
```

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 8,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "name" : "customer_name1",
          "email" : "修改email1",
          "customer_order" : {
            "name" : "customer"
          }
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "name" : "customer_name2",
          "email" : "email2",
          "customer_order" : {
            "name" : "customer"
          }
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "name" : "customer_name3",
          "email" : "email333",
          "customer_order" : {
            "name" : "customer"
          }
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "oid_1",
        "_score" : 1.0,
        "_routing" : "1",
        "_source" : {
          "order_id" : 1,
          "order_serial" : "serial_code1",
          "order_time" : "2021-08-04T13:13:02+08:00",
          "customer_order" : {
            "parent" : "1",
            "name" : "order"
          }
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "oid_2",
        "_score" : 1.0,
        "_routing" : "1",
        "_source" : {
          "order_id" : 2,
          "order_serial" : "serial_code2",
          "order_time" : "2021-08-04T13:14:43+08:00",
          "customer_order" : {
            "parent" : "1",
            "name" : "order"
          }
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "oid_3",
        "_score" : 1.0,
        "_routing" : "2",
        "_source" : {
          "order_id" : 3,
          "order_serial" : "serial_code3",
          "order_time" : "2021-08-04T13:16:17+08:00",
          "customer_order" : {
            "parent" : "2",
            "name" : "order"
          }
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "oid_4",
        "_score" : 1.0,
        "_routing" : "4",
        "_source" : {
          "order_id" : 4,
          "order_serial" : "serial_code4",
          "order_time" : "2021-08-05T02:04:29+08:00",
          "customer_order" : {
            "parent" : "4",
            "name" : "order"
          }
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "oid_5",
        "_score" : 1.0,
        "_routing" : "3",
        "_source" : {
          "order_id" : 5,
          "order_serial" : "serial_code5",
          "order_time" : "2021-08-05T02:05:38+08:00",
          "customer_order" : {
            "parent" : "3",
            "name" : "order"
          }
        }
      }
    ]
  }
}
```

可以看到，db中的数据都可能被同步到es中，在使用的过程中，我们只需要吧父子文档中的数据聚合起来就能实现db跨库查询了。





至此，canal到es的数据同步就告一段落。本篇文章的所有操作都是基于canal到mysql的tcp连接方式的基础上的，后续有时间会写一个加入mq的教程。