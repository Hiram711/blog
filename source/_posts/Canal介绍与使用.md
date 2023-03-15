title: Canal介绍与使用
author: 几时西风
tags:
  - canal
categories:
  - 数据同步
date: 2023-03-15 14:31:00
---
# Canal介绍与使用

## 简介

![upload successful](\blog\images\pasted-12.png)

**[canal](https://github.com/alibaba/canal) [kə'næl]**，译意为水道/管道/沟渠，主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费

早期阿里巴巴因为杭州和美国双机房部署，存在跨机房同步的业务需求，实现方式主要是基于业务 trigger 获取增量变更。从 2010 年开始，业务逐步尝试数据库日志解析获取增量变更进行同步，由此衍生出了大量的数据库增量订阅和消费业务。

基于日志增量订阅和消费的业务包括

- 数据库镜像
- 数据库实时备份
- 索引构建和实时维护(拆分异构索引、倒排索引等)
- 业务 cache 刷新
- 带业务逻辑的增量数据处理

当前的 canal 支持源端 MySQL 版本包括 5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x

**注意：本文中使用的canal版本为1.1.7版本**

**更多详细内容可以参考[Home · alibaba/canal Wiki (github.com)](https://github.com/alibaba/canal/wiki)**

## 工作原理

### MySQL主备复制原理


![upload successful](\blog\images\pasted-13.png)

- MySQL master 将数据变更写入二进制日志( binary log, 其中记录叫做二进制日志事件binary log events，可以通过 show binlog events 进行查看)
- MySQL slave 将 master 的 binary log events 拷贝到它的中继日志(relay log)
- MySQL slave 重放 relay log 中事件，将数据变更反映它自己的数据

### canal 工作原理

- canal 模拟 MySQL slave 的交互协议，伪装自己为 MySQL slave ，向 MySQL master 发送dump 协议
- MySQL master 收到 dump 请求，开始推送 binary log 给 slave (即 canal )
- canal 解析 binary log 对象(原始为 byte 流)

## 架构与概念

![upload successful](\blog\images\pasted-15.png)
- **canal是一个CS架构的程序，因此分为`client`与`server`,其中`server`可以包含多个`instance`**
- `server`：代表一个canal运行实例，对应于一个jvm，本质是一个基于SpringBoot的Web项目（包含Netty和Embeded两种实现），可以通过接口启动、配置、调度、监控instance
- `instance`：对应于一个数据队列，这个数据队列会完成解析binlog日志、binlog日志过滤、 binlog日志转储、位点元数据管理等核心功能,instance包含以下四个核心组件：
  - `eventParser` :数据源接入，模拟slave协议和master进行交互，协议解析
  - `eventSink` :Parser和Store链接器，进行数据过滤，加工，分发的工作
  - `eventStore` :数据存储，目前仅实现了Memory内存模式
  - `metaManager` :增量订阅&消费信息管理器
- `client`：消费数据的客户端，可以自己参考官方demo写java实现，也可以直接使用官方提供的工具包（CilentAdapter）



## 相关组件

![upload successful](\blog\images\pasted-16.png)

- canal.deployer:canal主体程序

- canal.admin:canal 的Web版管理后台，可以通过图形化界面管理配置参数,从而动态启停 `Server` 和 `Instance`，本文对此不多做深入研究

- canal.adapter:应当称为ClientAdapter,是在canal 1.1.1版本之后, 增加的客户端数据落地的适配及启动功能，目前支持功能:
  - 客户端启动器
  - 同步管理REST接口
  - 日志适配器, 作为DEMO
  - 关系型数据库的数据同步(表对表同步), ETL功能
  - HBase的数据同步(表对表同步), ETL功能
  - ElasticSearch多表数据同步,ETL功能

*另外还有集群HA的解决方案，但是目前暂时够用未深入研究*

**一般来说，用一个server带起若干instance，再加上adapter已经足够普通用户的使用**

## Server-QuickStart

### 准备

- 纯java开发，windows/linux均可支持

- jdk建议使用1.6.25以上的版本

- 对于自建 MySQL , 需要先开启 Binlog 写入功能，配置 binlog-format 为 ROW 模式，my.cnf 中配置如下

  ```
  [mysqld]
  log-bin=mysql-bin # 开启 binlog
  binlog-format=ROW # 选择 ROW 模式
  server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
  ```

  - 注意：针对阿里云 RDS for MySQL , 默认打开了 binlog , 并且账号默认具有 binlog dump 权限 , 不需要任何权限或者 binlog 设置,可以直接跳过这一步

- 授权 canal 链接 MySQL 账号具有作为 MySQL slave 的权限, 如果已有账户可直接 grant

  ```sql
  CREATE USER canal IDENTIFIED BY 'canal';  
  GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
  -- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
  FLUSH PRIVILEGES;
  ```



### 启动

- 下载 canal, 访问 [release 页面](https://github.com/alibaba/canal/releases) , 选择需要的包下载, 如以 1.0.17 版本为例

  ```bash
  wget https://github.com/alibaba/canal/releases/download/canal-1.0.17/canal.deployer-1.0.17.tar.gz
  ```

- 解压缩

  ```bash
  mkdir /tmp/canal
  tar zxvf canal.deployer-$version.tar.gz  -C /tmp/canal
  ```

  - 解压完成后，进入 /tmp/canal 目录，可以看到如下结构

    ```
    drwxr-xr-x 2 jianghang jianghang  136 2013-02-05 21:51 bin
    drwxr-xr-x 4 jianghang jianghang  160 2013-02-05 21:51 conf
    drwxr-xr-x 2 jianghang jianghang 1.3K 2013-02-05 21:51 lib
    drwxr-xr-x 2 jianghang jianghang   48 2013-02-05 21:29 logs
    ```

- 配置修改

  ```bash
  vi conf/example/instance.properties
  ```

  ```
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

  - canal.instance.connectionCharset 代表数据库的编码方式对应到 java 中的编码类型，比如 UTF-8，GBK , ISO-8859-1
  - 如果系统是1个 cpu，需要将 canal.instance.parser.parallel 设置为 false

- 启动

  ```bash
  sh bin/startup.sh
  ```

- 查看 server 日志

  ```bash
  vi logs/canal/canal.log</pre>
  ```

  ```
  2013-02-05 22:45:27.967 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## start the canal server.
  2013-02-05 22:45:28.113 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[10.1.29.120:11111]
  2013-02-05 22:45:28.210 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## the canal server is running now ......
  ```

- 查看 instance 的日志

  ```bash
  vi logs/example/example.log
  ```

  ```
  2013-02-05 22:50:45.636 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
  2013-02-05 22:50:45.641 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
  2013-02-05 22:50:45.803 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
  2013-02-05 22:50:45.810 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start successful....
  ```

  

- 关闭

  ```bash
  sh bin/stop.sh
  ```

### 踩坑

- base table不存在报错：[canal.deployer-1.1.6版本的安装包有bug #4245]([canal.deployer-1.1.6版本的安装包有bug · Issue #4245 · alibaba/canal (github.com)](https://github.com/alibaba/canal/issues/4245))
  
  ![upload successful](\blog\images\pasted-24.png)

  问题原因：

  为了增加新功能而导致的源码包与发布包不一致，且发布包有问题，具体信息可参考下面的链接，

  [兼容PolarDB-X的show databases返回 · Issue #4216 · alibaba/canal (github.com)](https://github.com/alibaba/canal/issues/4216)

  解决办法：

  - 自己重新下载源码重新打包depolyer，源码中已将此问题修复，但是发布的包并没有
  - 在instance.properties中应用如下设置过滤掉所有schema中的BASE TABLE表

  ```
  canal.instance.filter.black.regex=.*\\.BASE TABLE.*
  ```

  - 在instance.properties中应用如下设置只同步特定的表从而忽略BASE TABLE

  ```
  canal.instance.filter.regex=aglaia.tb_member,aglaia.tb_member_card
  ```

  

## Adapter

### 整体结构

client-adapter分为`适配器`和`启动器`两部分, 适配器为多个fat jar, 每个适配器会将自己所需的依赖打成一个包, 以SPI的方式让启动器动态加载, 目前所有支持的适配器都放置在plugin目录下

启动器为 SpringBoot 项目, 支持canal-client启动的同时提供相关REST管理接口, 运行目录结构为:

```
- bin
    restart.sh
    startup.bat
    startup.sh
    stop.sh
- lib
   ...
- plugin 
    client-adapter.logger-1.1.1-jar-with-dependencies.jar
    client-adapter.hbase-1.1.1-jar-with-dependencies.jar
    ...
- conf
    application.yml
    - hbase
        mytest_person2.yml
- logs
```

### 配置介绍

有两种方式可以对整个adapter进行配置，一种是使用总配置文件 application.yml，另一种则是使用bootstrap.yml将配置记录在数据库中

#### 总配置文件 application.yml

```yaml
canal.conf:
  canalServerHost: 127.0.0.1:11111          # 对应单机模式下的canal server的ip:port
  zookeeperHosts: slave1:2181               # 对应集群模式下的zk地址, 如果配置了canalServerHost, 则以canalServerHost为准
  mqServers: slave1:6667 #or rocketmq       # kafka或rocketMQ地址, 与canalServerHost不能并存
  flatMessage: true                         # 扁平message开关, 是否以json字符串形式投递数据, 仅在kafka/rocketMQ模式下有效
  batchSize: 50                             # 每次获取数据的批大小, 单位为K
  syncBatchSize: 1000                       # 每次同步的批数量
  retries: 0                                # 重试次数, -1为无限重试
  timeout:                                  # 同步超时时间, 单位毫秒
  mode: tcp # kafka rocketMQ                # canal client的模式: tcp kafka rocketMQ
  srcDataSources:                           # 源数据库
    defaultDS:                              # 自定义名称
      url: jdbc:mysql://127.0.0.1:3306/mytest?useUnicode=true   # jdbc url 
      username: root                                            # jdbc 账号
      password: 121212                                          # jdbc 密码
  canalAdapters:                            # 适配器列表
  - instance: example                       # canal 实例名或者 MQ topic 名
    groups:                                 # 分组列表
    - groupId: g1                           # 分组id, 如果是MQ模式将用到该值
      outerAdapters:                        # 分组内适配器列表
      - name: logger                        # 日志打印适配器
......           
```

说明:

1. 一份数据可以被多个group同时消费, 多个group之间会是一个并行执行, 一个group内部是一个串行执行多个outerAdapters, 比如例子中logger和hbase
2. 目前client adapter数据订阅的方式支持两种，直连canal server 或者 订阅kafka/RocketMQ的消息

#### 使用远程配置(Mysql)

1. 创建mysql schema

```sql
CREATE SCHEMA `canal_manager` DEFAULT CHARACTER SET utf8mb4 ;
```

2. 初始化数据

使用[canal_manager.sql](https://raw.githubusercontent.com/alibaba/canal/master/admin/admin-web/src/main/resources/canal_manager.sql)脚本建表并初始化Demo数据，其中canal_config表id=2的数据对应adapter下的application.yml文件，canal_adapter_config表对应每个adapter的子配置文件

3. 修改bootstrap.yml配置

```yaml
canal:
  manager:
    jdbc:
      url: jdbc:mysql://127.0.0.1:3306/canal_manager?useUnicode=true&characterEncoding=UTF-8
      username: root
      password: 121212
```

可以将本地application.yml文件和其他子配置文件删除或清空， 启动工程将自动从远程加载配置

修改mysql中的配置信息后会自动刷新到本地动态加载相应的实例或者应用

### QuickStart

1. 启动canal server

2. 修改conf/application.yml为:

   ```yaml
   server:
     port: 8081
   logging:
     level:
       com.alibaba.otter.canal.client.adapter.hbase: DEBUG
   spring:
     jackson:
       date-format: yyyy-MM-dd HH:mm:ss
       time-zone: GMT+8
       default-property-inclusion: non_null
   
   canal.conf:
     canalServerHost: 127.0.0.1:11111
     batchSize: 500                            
     syncBatchSize: 1000                       
     retries: 0                               
     timeout:                                 
     mode: tcp 
     canalAdapters:                            
     - instance: example                       
       groups:                                 
       - groupId: g1                           
         outerAdapters:                        
         - name: logger      
   ```

3. 启动

   ```bash
   bin/startup.sh
   ```

### adapter管理REST接口

1. 查询所有订阅同步的canal instance或MQ topic

```bash
curl http://127.0.0.1:8081/destinations
```

2. 数据同步开关

```bash
curl http://127.0.0.1:8081/syncSwitch/example/off -X PUT
```

针对 example 这个canal instance/MQ topic 进行开关操作. off代表关闭, instance/topic下的同步将阻塞或者断开连接不再接收数据, on代表开启

注: 如果在配置文件中配置了 zookeeperHosts 项, 则会使用分布式锁来控制HA中的数据同步开关, 如果是单机模式则使用本地锁来控制开关

3. 数据同步开关状态

```bash
curl http://127.0.0.1:8081/syncSwitch/example
```

查看指定 canal instance/MQ topic 的数据同步开关状态

4. 手动ETL

```bash
curl http://127.0.0.1:8081/etl/hbase/mytest_person2.yml -X POST -d "params=2018-10-21 00:00:00"
```

导入数据到指定类型的库, 如果params参数为空则全表导入, 参数对应的查询条件在配置中的etlCondition指定

5. 查看相关库总数据

```bash
curl http://127.0.0.1:8081/count/hbase/mytest_person2.yml
```



### 适配器列表

#### logger适配器

```
最简单的处理, 将受到的变更事件通过日志打印的方式进行输出, 如配置所示, 只需要定义name: logger即可
...
      outerAdapters:                        
      - name: logger 
```

#### Hbase适配器

同步HBase配置 : [Sync-HBase](https://github.com/alibaba/canal/wiki/Sync-HBase)

#### RDB适配器

同步关系型数据库配置 : [Sync-RDB](https://github.com/alibaba/canal/wiki/Sync-RDB)

目前内置支持的数据库列表:

1. MySQL
2. Oracle
3. PostgresSQL
4. SQLServer

使用了JDBC driver,理论上支持绝大部分的关系型数据库

#### ES适配器

同步关ES配置 : [Sync-ES](https://github.com/alibaba/canal/wiki/Sync-ES)

#### MongoDB适配器

#### Redis适配器

### RDB适配器与PostgreSQL配合使用

#### 操作步骤

1. 修改启动器配置conf/application.yml

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
  retries: 3
  timeout: 10
  accessKey:
  secretKey:
  consumerProperties:
    # canal tcp consumer
    canal.tcp.server.host: 127.0.0.1:11111
    canal.tcp.zookeeper.hosts:
    canal.tcp.batch.size: 500
    canal.tcp.username:
    canal.tcp.password:
    # kafka consumer
    kafka.bootstrap.servers: 127.0.0.1:9092
    kafka.enable.auto.commit: false
    kafka.auto.commit.interval.ms: 1000
    kafka.auto.offset.reset: latest
    kafka.request.timeout.ms: 40000
    kafka.session.timeout.ms: 30000
    kafka.isolation.level: read_committed
    kafka.max.poll.records: 1000
    # rocketMQ consumer
    rocketmq.namespace:
    rocketmq.namesrv.addr: 127.0.0.1:9876
    rocketmq.batch.size: 1000
    rocketmq.enable.message.trace: false
    rocketmq.customized.trace.topic:
    rocketmq.access.channel:
    rocketmq.subscribe.filter:
    # rabbitMQ consumer
    rabbitmq.host:
    rabbitmq.virtual.host:
    rabbitmq.username:
    rabbitmq.password:
    rabbitmq.resource.ownerId:
  srcDataSources: #导出数据的mysql数据源配置，后续会在表映射配置文件中使用到
    defaultDS:
      url: jdbc:mysql://127.0.0.1:3306/mytest?useUnicode=true
      username: root
      password: 121212
  canalAdapters:
  - instance: example # canal instance Name or mq topic name
    groups:
    - groupId: g1
      outerAdapters:
      - name: logger
      - name: rdb # 指定为rdb类型同步
        key: postgres1  # 指定adapter的唯一key, 与表映射配置文件中outerAdapterKey对应
        properties: #目标数据库的链接信息
          jdbc.driverClassName: org.postgresql.Driver # jdbc驱动名, 部分jdbc的jar包需要自行放致lib目录下
          jdbc.url: jdbc:postgresql://localhost:5432/postgres # jdbc url
          jdbc.username: postgres # jdbc username
          jdbc.password: 121212 # jdbc password
          threads: 1  # 并行执行的线程数, 默认为1
          commitSize: 3000
```

2. 修改 conf/rdb/mytest_user.yml文件

   ```yaml
   dataSourceKey: defaultDS        # 源数据源的key, 对应上面配置的srcDataSources中的值
   destination: example            # cannal的instance或者MQ的topic
   groupId: g1                     # 对应MQ模式下的groupId, 只会同步对应groupId的数据
   outerAdapterKey: postgres1      # adapter key, 对应上面配置outAdapters中的key
   concurrent: true                # 是否按主键hash并行同步, 并行同步的表必须保证主键不会更改及主键不能为其他同步表的外键!!
   dbMapping:
     database: mytest              # 源数据源的database/shcema
     table: user                   # 源数据源表名
     targetTable: ods.tb_user   # 目标数据源的模式名.表名
     targetPk:                     # 主键映射
       id: id                      # 如果是复合主键可以换行映射多个
     #mapAll: true                 # 是否整表映射, 要求源表和目标表字段名一模一样 (如果targetColumns也配置了映射,则以targetColumns配置为准)
     targetColumns:                # 字段映射, 格式: 目标表字段: 源表字段, 如果字段名一样源表字段名可不填
       id:
       name:
       role_id:
       c_time:
       test1: 
     #etlCondition: "where c_time>={}"
     commitBatch: 3000 # 批量提交的大小
   ```

3. 启动RDB程序

   - 将目标库的jdbc jar包放入lib文件夹, 这里放入ojdbc6.jar (如果是其他数据库则放入对应的驱动)

   - 启动canal-adapter启动器

     ```bash
     bin/startup.sh
     ```

   - 全量同步一遍数据

     ```bash
     curl http://127.0.0.1:8081/etl/rdb/mytest_user.yml -X POST
     ```

   - 确认RDB是否增长运行

     ```bash
     curl http://127.0.0.1:8081/destinations
     ```

#### 踩坑

- 全量同步报错：

  ```
  ERROR c.a.otter.canal.client.adapter.rdb.service.RdbEtlService - org.postgresql.util.PSQLException: Fetch size must be a value greater to or equal to 0.
  java.lang.RuntimeException: org.postgresql.util.PSQLException: Fetch size must be a value greater to or equal to 0.
  ```

  问题原因：

  错误设置postgres数据库的FetchSize

  [postgres全量同步报错 · Issue #2146 · alibaba/canal (github.com)](https://github.com/alibaba/canal/issues/2146)

  解决办法：

  修改client-adapter/common/src/main/java/com/alibaba/otter/canal/client/adapter/support/Util.java文件第41行和第55行，如下图所示
  
  ![upload successful](\blog\images\pasted-19.png)

  **此问题目前只能通过此方式解决，需要自己自己修改源码后重新打包**

- 同步报错：某字段 not matched

  问题原因：

  sql关键字错误，误认为所有数据库的表名都可以用mysql下列形式表示，导致字段全都无法匹配

  ```
  `dbname`.`tablename`
  ```

  [adapter 同步到目标库 报错Target column: id not matched · Issue #2941 · alibaba/canal (github.com)](https://github.com/alibaba/canal/issues/2941)

  经调查，根据issue内的方法直接改client-adapter/rdb/src/main/java/com/alibaba/otter/canal/client/adapter/rdb/support/SyncUtil.java中的getDBTableName和getBacktickByDbType方法是无用的，因为该方法其实并没有错误如下图
  ![upload successful](\blog\images\pasted-21.png)
   需要修改的是client-adapter/rdb/src/main/java/com/alibaba/otter/canal/client/adapter/rdb/service/RdbEtlService.java中的executeSqlImport方法，此方法中错误的将需要使用目标数据源的地方用了源头数据源，导致了关键字全部按照mysql的方法进行拼接，最终导致错误，因此需要做如下修改：
  
  ![upload successful](\blog\images\pasted-22.png)
  
  ![upload successful](\blog\images\pasted-23.png)

  做完以上修改后，重新运行如下打包命令,然后在项目根目录的target文件夹下找到对应的包重新部署：

  ```bash
  mvn clean install -Dmaven.test.skip -Denv=release
  ```

  

  

