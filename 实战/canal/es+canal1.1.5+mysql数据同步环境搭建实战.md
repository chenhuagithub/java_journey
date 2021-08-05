# canal+es+mysql数据同步环境搭建

canal的基本介绍在这里就不再阐述了，有兴趣看官网的权威级别的介绍：https://github.com/alibaba/canal

下面直接进入环境搭建的环节

### mysql配置

1. 首先我们需要对mysql进行配置

```mysql
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
```

`bingo-format=ROW`的意思是mysql每一行数据更改都会产生一个biglog

2. 授权 canal 链接 MySQL 账号具有作为 MySQL slave 的权限, 如果已有账户可直接 grant

```mysql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

---

### canal

canal有客户端和服务端之分，客户端的包名叫`canal.adapter`,服务端的包名叫`canal.deployer`,`canal.deployer`的作用主要是充当mysql的slave角色，订阅mysql的增量binlog；`canal.adapter`的作用主要是解析`canal.deployer`发送过来的数据，并把数据同步到es中，当然，`canal.deployer`不仅仅可以同步数据到es中，还可以同步数据到rdb或者hbase等，我们这里只讨论es的情况。

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803174815.png)

#### 启动canal.deployer

1. 首先我们要把canal.deployer下载下来，下载地址为https://github.com/alibaba/canal/releases。

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803175053.png)

2. 解压下载下来的压缩包

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803175402.png)

3. 修改配置

```vim
vim conf/example/instance.properties
```

```sh
## mysql serverId
canal.instance.mysql.slaveId = 1234
#position info，需要改成自己的数据库信息
canal.instance.master.address = 127.0.0.1:3306 
canal.instance.master.journal.name = 
canal.instance.master.position = 
canal.instance.master.timestamp = 
#canal.instance.standby.address = 
#canal.instance.standby.journal.name =
#canal.instance.standby.position = 
#canal.instance.standby.timestamp = 
#username/password，需要改成自己的数据库信息
canal.instance.dbUsername = canal  
canal.instance.dbPassword = canal
canal.instance.defaultDatabaseName =
canal.instance.connectionCharset = UTF-8
#table regex
canal.instance.filter.regex = .\*\\\\..\*
```

注意:

- canal.instance.connectionCharset 代表数据库的编码方式对应到 java 中的编码类型，比如 UTF-8，GBK , ISO-8859-1
- 如果系统是1个 cpu，需要将 canal.instance.parser.parallel 设置为 false

4. 启动

```sh
sh bin/startup.sh
```

5. 查看 server 日志

```sh
vi logs/canal/canal.log
```

```sh
2013-02-05 22:45:27.967 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## start the canal server.
2013-02-05 22:45:28.113 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[10.1.29.120:11111]
2013-02-05 22:45:28.210 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## the canal server is running now ......
```

6. 查看 instance 的日志

```sh
vi logs/example/example.log
```

```sh
2013-02-05 22:50:45.636 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
2013-02-05 22:50:45.641 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
2013-02-05 22:50:45.803 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
2013-02-05 22:50:45.810 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start successful....
```

7. 关闭

```sh
sh bin/stop.sh
```



#### 启动canal.adpater

1. 下载canal adapter

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803193146.png)

一定要下载快照版的adpter；一定要下载快照版的adpter；一定要下载快照版的adpter；

别问，问就是v1.1.5版的adapter有bug。

2. 解压

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803193542.png)

3. 修改conf/application.yml

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
```

4. 启动

```sh
bin/startup.sh
```

5. 在mytest数据库写入数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803194715.png)

6. 查看日志

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803195015.png)

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803194849.png)

可以看到，canal adapter成功接收到mysql的更改数据。

### es配置

es的配置这里不展开讲，默认已经安装好es环境，下面主要讨论canal adapter是如何配置数据的同步～

1. 修改conf/application.yml

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
  # 数据库配置
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
          # security.auth: test:123456 #  only used for rest mode#
          cluster.name: docker-cluster

```

2. 编写映射文件

映射文件在canal-adapter项目下的es7文件夹中

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803195655.png)

mytest_user.yml文件内容如下:

```xml
dataSourceKey: defaultDS
destination: example
groupId: g1
esMapping:
  _index: mytest_user
  _id: _id
#  upsert: true
#  pk: id
  sql: "select a.id as _id, a.name, a.role_id, b.role_name,
        a.c_time from user a
        left join role b on b.id=a.role_id"
#  objFields:
#    _labels: array:;
  etlCondition: "where a.c_time>={}"
  commitBatch: 3000

```

上面user表和role表设计如下：

```mysql
CREATE TABLE `user` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL,
  `role_id` int(11) DEFAULT NULL,
  `c_time` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;
```

```mysql
CREATE TABLE `role` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `role_name` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;
```

配置到这里先别着急启动`canal adapter`，我们还要事先在es建好相应的索引

3. es创建索引

```json
PUT /mytest_user
{
  "mappings":{
    "properties":{
      "name":{"type": "keyword"},
      "role_id":{"type": "integer"},
      "role_name":{"type": "keyword"},
      "c_time":{"type": "date"}
    }
  }
}
```

4. 启动canal adapter

```sh
bin/startup.sh
```

5. 在数据库中插入数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803200324.png)

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803200408.png)

6. 查看es中的索引数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210803201123.png)

至此，整个环境已经搭建成功了！！！

讲到这里，有人可能就有疑问了，我特喵每次都只能修改编译后的配置文件，而且只能通过staterup.sh文件启动程序，还能不能当个人？如果你有这个困惑，别急，下面开始解决你的困惑。



其实，所有的编译后的文件都是从github上的源码文件编译而来的，我们只需要把整个github项目拉取下来就能唯我所用。

### canal项目启动

1. 拉取项目

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804101116.png)

选择版本，我这里选择的是`canal-1.1.5-alpha-2`的版本，下载下来后解压，并用idea打开

2. mvn install

在idea打开后，把整个项目install一下，一定要install一下，否则项目启动会报错！！！

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804101536.png)

install完后，我们就可以正常启动项目了，该项目的`client-adapter`模块对应上面所说的`canal.adapter`,	而`deployer`对应上面所说的`canal.deployer`。

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804102025.png)

值得注意的是，deployer项目没有入口程序，只能编译源代码后通过官方提供的startup.sh文件启动。

3. 启动client-adapter项目

我们在launcher模块和es7模块配置好对应的配置文件后，通过CanalAdapterApplication这个入口类启动程序。

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804103342.png)

4. 测试

接下来，我们在数据库修改一个数据

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804103508.png)

不出意外的话，你会在idea的控制台看到数据库的数据更改日志

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804103810.png)

我们去es看看数据有没有同步过来～

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210804103943.png)

可以看到，数据已经成功同步到es之中了。



至此，canal相关环境的搭建已经基本拿下了，接下来的任务就是研究canal adapter的使用了。