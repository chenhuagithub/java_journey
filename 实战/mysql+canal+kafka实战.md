# Mysql + Canal + Kafka环境搭建实战

​	近日在公司接了一个新需求，实现一个系统的报表功能，由于报表可能会牵涉到n张表，如果直接使用Mysql实现的话会对DB造成很大的压力，因此公司选用ES作为报表实现的平台，并使用阿里巴巴的开源产品Canal实现数据同步，并使用Kafuka消息队列实现Mysql的binlog异步获取。

​	Mysql+Canal+Kafka的环境搭建花费了我一天的时间，中途有很多bug和版本问题，因此打算记录一下环境搭建的过程～



## Mysql配置

> 首先我们需要对Mysql进行简单的配置，Mysql的具体安装下面就不阐述了

我们在mysql的配置文件中找到`my.cnf`这个文件

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618100532.png)

使用vim设置如下信息：

```mysql
[mysqld]
# 打开binlog
log-bin=mysql-bin
# 选择ROW(行)模式，意味着mysql只要一行数据改变都会产生一个binlog
binlog-format=ROW
# 配置MySQL replaction需要定义，不要和canal的slaveId重复
server_id=1
```

配置完成后，重启mysql并使用命令查看是否打开binlog模式：

```mysql
show variables like 'log_bin';
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618101220.png)

查看binlog当前的日志列表：

```mysql
show binary logs ;
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618100954.png)

查看当前正在写入的binlog文件：

```mysql
show master status ;
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618101108.png)

至此，Mysql的配置已经完成，接下来开始安装和配置Canal



## Canal配置

在配置Canal之前，我们需要先安装Canal，这里我使用的docker安装的方式～

拉取镜像：

```shell
docker pull canal/canal-server:v1.1.5
```

查看镜像：

```shell
docker images
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618101714.png)

启动canal容器：

```shell
docker run --name canal -p 11111:11111 -d canal/canal-server:v1.1.5
```

查看容器是否已经启动：

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618101946.png)

可以看到canal已经成功启动，接下来，我们需要进入canal容器中进行一些配置

进入canal容器：

```shell
docker exec -it b91a /bin/bash
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618102214.png)

进入到`/home/admin/canal-server/conf/example`目录

```
cd /home/admin/canal-server/conf/example
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618102419.png)

编辑`instance.properties`文件并做如下的配置：

```properties
# position info
# 配置自己对应的mysql连接
canal.instance.master.address=172.22.50.254:3306
# username/password
# 配置mysql授权的用户名和密码
canal.instance.dbUsername=canal
canal.instance.dbPassword=12345678
```

其他的配置使用系统默认就行，不需要配置。

然后推出canal容器并重启canal容器

```
docker restart b91a
```

接下来我们使用maven项目来测试一下canal的配置效果

导入canal的依赖

```xml
<dependency>
  <groupId>com.alibaba.otter</groupId>
  <artifactId>canal.client</artifactId>
  <version>1.1.5</version>
</dependency>
```

编写canal client客户端代码

// todo



## Kafaka配置

Kafka是由zookeeper管理的，所以我们需要同时安装Kafka和Zookeeper。同样，我们使用dokcer的方式进行安装

拉取Zookeeper镜像：

```shell
docker pull wurstmeister/zookeeper
```

启动zk镜像：

```shell
docker run -d --name zookeeper -p 2181:2181 wurstmeister/zookeeper
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618111944.png)

拉取Kafka镜像：

```shell
docker pull wurstmeister/kafka
```

启动Kafka容器：

```shell
docker run -d --name kafka -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=172.22.50.254:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.22.50.254:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka
```

注意：`172.22.50.254`需要换成自己电脑的ip地址

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618112251.png)

现在我们来测试Kafka+zk是否搭建成功～

进入Kafka容器

```shell
docker exec -it 6d90 /bin/sh
```

接着进入`/opt/kafka_2.13-2.7.0/bin`目录

```shell
cd /opt/kafka_2.13-2.7.0/bin
```

运行kafka生产者发送消息

```shell
./kafka-console-producer.sh --broker-list localhost:9092 --topic sun（消息订阅的名字）
```

并录入消息（这里以hello为例子）

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618112858.png)

接下来，运行kafka消费者接收消息

```shell
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic sun --from-beginning
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618113007.png)

可见，消息消费成功，即Kafka环境搭建成功

我们再来看看zk的环境搭建情况

进入zk容器

```shell
docker exec -it 2292 /bin/sh
```

进入`/opt/zookeeper-3.4.13/bin`目录

```shell
cd /opt/zookeeper-3.4.13/bin
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618113342.png)

执行**./zkCli.sh**启动zk客户端：

```shell
./zkCli.sh
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618113436.png)

查看zk的节点情况：

```shell
ls /brokers/topics
```

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618113542.png)

可以看到，有我们上面所使用的订阅节点（sun），说明zk也是没有问题的。

接下来，我们可以在canal中配置Kafka，让Kafka直接接受mysql的binlog文件

## Canal配置Kafka



```properties
canal.destinations = example #这里配置开启的 instance
canal.serverMode = kafka 	# 更改模式，直接把数据扔进 Kafka
kafka.bootstrap.servers = 172.22.50.254:9092 # Kafka 的地址
```

说明`canal.destinations`的值与canal的conf文件夹下的目录名一致：

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618114332.png)

接着我们需要配置`example`实例下的`instance.properties`文件

![](https://gitee.com/chenhuagitee/image-bed/raw/master/img/20210618114558.png)

红色矩形圈起来的是Kafka里面创建的消息订阅名称，如果Kafka中没有该名称的话一定要先到Kafka中创建该名称。

至此，在canal中有关Kafka的配置已经配置完成，接下来就是用springboot来测试我们的配置成果了～

引入kafka依赖

```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
```

编写kafka消息监听器：

```java
@Component
@Log4j2
public class KafkaConsumerListener {

    @KafkaListener(topics = "canal")
    public void onMessage1(String message){
        System.out.println(message);
        log.info("kafka-topic1接收结果:{}",message);
    }
}
```

启动springboot项目！

现在，我们来随意创建一张数据表：

```mysql
CREATE TABLE `tb_commodity_info` (
    `id` varchar(32) NOT NULL,
    `commodity_name` varchar(512) DEFAULT NULL COMMENT '商品名称',
    `commodity_price` varchar(36) DEFAULT '0' COMMENT '商品价格',
    `number` int(10) DEFAULT '0' COMMENT '商品数量',
    `description` varchar(2048) DEFAULT '' COMMENT '商品描述',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品信息表';
```

此时控制台会打印kafka接受到的消息：

```json
{"data":null,"database":"test","es":1623988327000,"id":4,"isDdl":true,"mysqlType":null,"old":null,"pkNames":null,"sql":"/* ApplicationName=IntelliJ IDEA 2020.1 */ CREATE TABLE `tb_commodity_info` (\n    `id` varchar(32) NOT NULL,\n    `commodity_name` varchar(512) DEFAULT NULL COMMENT '商品名称',\n    `commodity_price` varchar(36) DEFAULT '0' COMMENT '商品价格',\n    `number` int(10) DEFAULT '0' COMMENT '商品数量',\n    `description` varchar(2048) DEFAULT '' COMMENT '商品描述',\n    PRIMARY KEY (`id`)\n) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品信息表'","sqlType":null,"table":"tb_commodity_info","ts":1623988327851,"type":"CREATE"}

```

可见，Kafka能够正确的接受到canal发送过来的binlog了，我们再继续向该表中插入一条数据

```mysql
INSERT INTO tb_commodity_info VALUES('4e81b81ad8e881aabed600163e046cc2','叉烧包','3.99',3,'又大又香的叉烧包，老人小孩都喜欢');
```

控制台打印：

```json
{"data":[{"id":"4e81b81ad8e881aabed600163e046cc2","commodity_name":"叉烧包","commodity_price":"3.99","number":"3","description":"又大又香的叉烧包，老人小孩都喜欢"}],"database":"test","es":1623988425000,"id":5,"isDdl":false,"mysqlType":{"id":"varchar(32)","commodity_name":"varchar(512)","commodity_price":"varchar(36)","number":"int(10)","description":"varchar(2048)"},"old":null,"pkNames":["id"],"sql":"","sqlType":{"id":12,"commodity_name":12,"commodity_price":12,"number":4,"description":12},"table":"tb_commodity_info","ts":1623988425520,"type":"INSERT"}
```

根据控制台的打印，我们可以得到mysql数据库中插入的数据是什么～



到此，环境已经搭建成功，kafka能够获取到binlog数据，后期接入es后，在业务层直接根据kafka中的数据更新es即可～