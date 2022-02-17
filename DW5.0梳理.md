[toc]

# xsync&集群配置

1. rsync 目录分发，rsync -alvz /root -> 这样改变目录

2. rsync -alvz /root/ -> 这样会递归改变目录下的所有文件
3. -a 归档 -l 改变连接 -v 详细模式输出 -z 对备份的文件在传输时进行压缩处理

4. [xsync文件](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/xsync.sh)

# mysql 密码策略

```sql
INSTALL PLUGIN validate_password SONAME 'validate_password.dll';(windows后缀为dll,linux后缀为so)
show variables like "validate_password%";
set global validate_password_policy=Low;
set global validate_password_mixed_case_count=0;
set global validate_password_length=2;
```

# 日志采集flume

系统数据流程设计
<span style="text-align: center;display:block;">
<img src="./img/系统数据流程设计.png"/></span>

1. 日志服务器 -> 本地磁盘 -> flume监控(logfile->拦截器->kafkachannel) -> kafka集群 -> flume监控(kafkasouce -> filechannel -> hdfs) -> hdfs

2. vim [logfile_flume_kafka.conf](https://github.com/Ciaran-crop/conf/tree/master/DWConfiguration/logfile_flume_kafka.conf)
3. 编写IDEA [CollectInterceptor](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/collect-interceptor-1.0-SNAPSHOT-jar-with-dependencies.jar)
4. vim [kafka_flume_hdfs.conf](https://github.com/Ciaran-crop/conf/tree/master/DWConfiguration/kafka_flume_hdfs.conf)
5. 启动kafka,flume [相关文件f1.sh](https://github.com/Ciaran-crop/conf/tree/master/DWConfiguration/f1.sh) [相关文件f2.sh](https://github.com/Ciaran-crop/conf/tree/master/DWConfiguration/f2.sh)

# 解决hive乱码

1. 修改hive库中的comment 字符编码

```sql
use hive;
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8;
alter table TABLE_PARAMS modify column PARAM_VALUE mediumtext character set utf8;
```

2. 修改 `hive-site.xml`

```xml
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://aliyun101:3306/metastore?useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
</property>
```

# DataX

0. 高可用，需要将hdfs-site.xml,core-site.xml,hive-site.xml放入hdfs-writer.jar 或hdfs-reader.jar 中

1. 启动命令 `python bin/datax.py job/job.json` 

2. 获得配置文件 `python bin/datax.py -r mysqlreader -w myhdfswriter > myjob.json`

3. 参数传递 `python bin/datax.py -p"-Ddt=2020-06-14" job/myjob.json` [myjob.json](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/myjob.json)


# mysql 主从复制

1. vim /etc/my.cnf 并重启mysql
```
#数据库id
server-id = 1
#启动binlog，该参数的值会作为binlog的文件名
log-bin=mysql-bin
#binlog类型，maxwell要求为row类型
binlog_format=row
#启用binlog的数据库，需根据实际情况作出修改
binlog-do-db=gmall
```

# maxwell

1. 创建数据库

```sql
CREATE DATABASE maxwell;
CREATE USER 'maxwell'@'%' IDENTIFIED BY 'maxwell';
GRANT ALL ON maxwell.* TO 'maxwell'@'%';
GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
```

2. vim config.properties

```
#Maxwell数据发送目的地，可选配置有stdout|file|kafka|kinesis|pubsub|sqs|rabbitmq|redis
producer=kafka
#目标Kafka集群地址
kafka.bootstrap.servers=aliyun101:9092,aliyun102:9092,aliyun103:9092
#目标Kafka topic，可静态配置，例如:maxwell，也可动态配置，例如：%{database}_%{table}
kafka_topic=maxwell

#MySQL相关配置
host=aliyun101
user=maxwell
password=maxwell
jdbc_options=useSSL=false&serverTimezone=Asia/Shanghai
```

3. maxwell 启动 `maxwell --config ${MAXWELL_HOME}/config.properties --daemon` [maxwell.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/mxwell.sh)

# 全量同步

编写脚本
1、json配置文件生成  vim [gen_import_config.py](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/gen_import_config.py)   `yum install -y MySQL-python` 
2、json配置文件生成 vim [gen_import_config.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/gen_import_config.sh)
3、datax同步脚本 vim [mysql_to_hdfs_full.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/mysql_to_hdfs_full.sh)

# 增量同步

0. 增量表首日全量同步 -> mysql binlog(文件名log-bin=mysql-binxxx.log) -> maxwell -> kafka -> flume(kafkasource->filechannel->hdfs) -> hdfs

1. 增量表首日全量同步：使用bootstrap,输出的type有bootstrap-start,bootstrap-insert,bootstrap-end，其中只有bootstrap-insert类型的json有数据  vim [mysql_to_kafka_inc_init.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/mysql_to_kafka_inc_init.sh)

1. vim config.properties(maxwell)

```
kafka_topic=%{table}
#表过滤，只同步特定的13张表
filter= include:gmall.cart_info,include:gmall.comment_info,include:gmall.coupon_use,include:gmall.favor_info,include:gmall.order_detail,include:gmall.order_detail_activity,include:gmall.order_detail_coupon,include:gmall.order_info,include:gmall.order_refund_info,include:gmall.order_status_log,include:gmall.payment_info,include:gmall.refund_payment,include:gmall.user_info
```

说明：filter填inclue:库名.表名,... 表示要同步的表

2. 重新启动maxwell
3. vim [kafka_to_hdfs_db.conf](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/kafka_to_hdfs_db.conf)
4. 启动kafka，maxwell，flume。[相关文件f3.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/f3.sh)
5. 增量表首日全量同步
6. 保持通道开启 

# 数据仓库(有难点)

1. 数据仓库中积累的大量历史数据对于数据科学家和业务分析师也是十分宝贵的

数仓核心架构
<span style="text-align: center;display:block;">
<img src="./img/数仓核心架构.png"/></span>

2. 数据模型使得 数据使用 

    高性能：良好的数据模型能够帮助我们快速查询所需要的数据。
    低成本：良好的数据模型能减少重复计算，实现计算结果的复用，降低计算成本。
    高效率：良好的数据模型能极大的改善用户使用数据的体验，提高使用数据的效率。
    高质量：良好的数据模型能改善数据统计口径的混乱，减少计算错误的可能性。

## 数据仓库建模方法论

1. ER模型(实体关系模型)

    实体关系模型将复杂的数据抽象为两个概念——实体和关系。实体表示一个对象，例如学生、班级，关系是指两个实体之间的关系，例如学生和班级之间的从属关系。

2. 数据库规范化

    目的是减少数据冗余，增强数据的一致性。

3. 三范式

* 第一范式：属性不可切割
* 第二范式：不存在非主属性对主属性的部分函数依赖。如分数 依赖（学号，课名），但是姓名并不完全依赖于（学号，课名）
* 第三范式：不存在非主属性对主属性的传递函数依赖。如学号->系名->系主任


## 维度模型(重点，难点)

通过事实和维度两个概念进行呈现。
维度建模以数据分析作为出发点，为数据分析服务，因此它关注的重点的用户如何更快的完成需求分析以及如何实现较好的大规模复杂查询的响应性能

### 事实表

1. 事实表作为数据仓库维度建模的核心，紧紧围绕着业务过程来设计。
2. 包含与该业务过程有关的维度引用（**维度表外键**）以及该业务过程的**度量**（通常是**可累加的数字类型字段**）
3. 特点：细长
4. 分类：事务事实表、周期快照事实表和累积快照事实表

#### 事务型事实表

1. 记录各业务过程，它保存的是各业务过程的**原子操作事件**
2. **设计流程**： 选择业务过程→声明粒度→确认维度→确认事实
3. 典型的粒度声明（**粒度最小**）：订单事实表中一行数据表示的是一个订单中的一个商品项。
4. **维度的丰富程度**就决定了维度模型能够支持的指标丰富程度
5. “事实”一词，指的是**每个业务过程的度量值**（通常是可累加的数字类型的值，例如：次数、个数、件数、金额等）。
6. *不足*（不适合的指标）：存量型指标(适合周期快照事实表)，多事务关联统计（适合累计事实表）

#### 周期型快照事实表

1. 周期快照事实表以具有**规律性的、可预见的时间间隔**来记录事实
2. 分析一些存量型（**规律性的、可预见的时间间隔**）（例如商品库存，账户余额）或者状态型（**连续性的**）（空气温度，行驶速度）指标
3. **设计流程**：确定粒度 -> 确认事实
4. 确认粒度即 **确定采样周期和维度**。采样周期通常选择每日
5. 确认事实，根据统计指标决定。如库存，余额等
6. 事实类型：可加事实（所有维度进行累加），半可加事实（一部分维度进行累加），不可加事实（不具备可加性，例如比率型事实）。

#### 累积型快照事实表

1. 累积型快照事实表通常具有多个日期字段，每个日期对应**业务流程**中的一个关键业务过程（里程碑）。
2. 选择业务过程→声明粒度→确认维度→确认事实
3. 选择一个业务流程中需要关联分析的多个关键业务过程
4. 每行数据表示的是什么，尽量选择最小粒度
5. 每各业务过程均需要一个日期维度

### 维度表

1. 维度表是维度建模的基础和灵魂
2. 维度表主要包含一个主键和各种维度字段，维度字段称为维度属性

3. **设计步骤**：确定维度-- -> 确定主维表和相关维表 -> 确定维度属性
4. 在设计事实表时，已经确定了与每个事实表相关的维度。如果某些维度表的维度属性很少，则可不创建该维度表，而把该表的维度属性直接增加到与之相关的事实表中，这个操作称为**维度退化**
5. 此处的主维表和相关维表均指业务系统中与某维度相关的表
6. 确定维度属性：尽可能生成**丰富的维度属性**。尽量不使用编码，而使用明确的文字说明，一般可以**编码和文字共存**。尽量**沉淀出通用的维度属性**
7. 维度变化：全量快照表（每天同步一次）。拉链表（记录信息生命周期）。
8. 多值维度：事实表一条记录对应维度表多条记录。说明事实表粒度**不够小**。
9. 多值属性：维表中的某个属性同时有多个值，称之为“多值属性”。将多值属性**放到一个字段**

## 数据仓库设计(重点，难点)

1. 数据仓库分层规划
<span style="text-align: center;display:block;">
<img src="./img/数据仓库分层规划.png"/></span>

2. 数据仓库构建流程
<span style="text-align: center;display:block;">
<img src="./img/数据仓库构建流程.png"/></span>

### 数据调研(有重点)

#### 业务调研（在建表时有用）

0. 需要建立什么样的表

1. 主要目标是熟悉业务流程、熟悉业务数据
2. 熟悉业务流程要求做到，明确每个业务的具体流程，需要将该业务所包含的**每个业务过程一一列举出来**
3. 熟悉业务数据要求做到，将数据（包括埋点日志和业务数据表）与**业务过程对应起来**

#### 需求分析（在指标时有用）

0. 需要统计什么

1. 典型的需求指标如，最近一天各省份手机品类订单总额
2. 分析需求时，需要明确需求所需的**业务过程及维度**，例如该需求所需的业务过程就是买家下单，所需的维度有日期，省份，商品品类。
3. 做完业务分析和需求分析之后，要保证**每个需求都能找到与之对应的业务过程及维度**。
4. **指标体系**：**需求 -> 依赖指标 -> 原子指标;通过统计的到用的比较多的公共指标，构建DWS层**

### 明确数据域

1. 数据域的意义是便于数据的管理和应用

|数据域|业务过程|
| :-- | :-- |
|交易域|加购、下单、取消订单、支付成功、退单、退款成功|
|流量域|页面浏览、启动应用、动作、曝光、错误|
|用户域|注册、登录|
|互动域|收藏、评价|
|工具域|优惠券领取、优惠券使用（下单）、优惠券使用（支付）|

### 构建业务总线矩阵

矩阵的行是一个个业务过程，矩阵的列是一个个的维度，行列的交点表示业务过程与维度的关系

<span style="text-align: center;display:block;">
<img src="./img/业务总线矩阵.png"/></span>

### 明确统计指标

* 原子指标：基于某一业务过程的度量值。包含三要素，分别是**业务过程、度量值和聚合逻辑**。订单总额就是一个典型的原子指标

* 派生指标：基于原子指标。原子指标 + 统计周期（时间范围） + 业务限定（where语句） + 统计粒度（group by）
* 衍生指标：基于派生指标。由一个或多个派生指标复合而成。如比例性指标

* 当统计需求足够多时，必然会出现部分统计需求对应的派生指标相同的情况。这种情况下，我们就可以考虑将这些公共的派生指标保存下来，这样做的主要目的就是减少重复计算，提高数据的复用性

### 维度模型设计

参照上述得到的业务总线矩阵即可。事实表存储在DWD层，维度表存储在DIM层

### 汇总模型设计

* 一张汇总表通常包含**业务过程相同、统计周期相同、统计粒度相同**的多个派生指标

* 一个事实表对应多个汇总表


## 数据仓库环境准备(有难点)

1. Hive环境搭建

* Hive on Spark：Hive既作为存储元数据又负责SQL的解析优化，**语法是HQL语法**，执行引擎变成了Spark，Spark负责采用RDD执行。
* Spark on Hive : Hive只作为存储元数据，Spark负责SQL解析优化，**语法是Spark SQL语法**，Spark负责采用RDD执行。

2. Hive on Spark配置（难点）

官网下载的Hive3.1.2和Spark3.0.0默认是不兼容的。因为Hive3.1.2支持的Spark版本是2.4.5，所以需要我们重新编译Hive3.1.2版本。
**编译步骤**：官网下载Hive3.1.2源码，修改pom文件中引用的Spark版本为3.0.0，如果编译通过，直接打包获取jar包。如果报错，就根据提示，修改相关方法，直到不报错，打包获取jar包。

3. 向HDFS上传Spark jar包,Spark任务资源分配由Yarn来调度.Spark的依赖上传到HDFS集群路径，这样集群中任何一个节点都能获取到。

4. 修改hive-site.xml文件

```xml
<!--Spark依赖位置（注意：端口号8020必须和namenode的端口号一致）-->
<property>
    <name>spark.yarn.jars</name>
    <value>hdfs://nameservice1/spark-jars/*</value>
</property>
  
<!--Hive执行引擎-->
<property>
    <name>hive.execution.engine</name>
    <value>spark</value>
</property>

```

5. 配置SPARK_HOME环境变量

6. 在hive中创建spark配置文件

```
spark.master                               yarn
spark.eventLog.enabled                   true
spark.eventLog.dir                        hdfs://nameservice1/spark-history
spark.executor.memory                    1g
spark.driver.memory					   1g
```

7. Yarn环境配置

8. 增加ApplicationMaster资源比例。容量调度器对每个资源队列中同时运行的Application Master占用的资源进行了限制 `vim capacity-scheduler.xml`

```xml
<property>
    <name>yarn.scheduler.capacity.maximum-am-resource-percent</name>
    <value>0.8</value>
</property>
```

9. 数据仓库开发环境。使用DBeaver或者DataGrip

## 数仓开发之ODS层

1. ODS层的设计要点如下：

    1. ODS层的表结构设计依托于从**业务系统同步过来的数据结构**

    2. ODS层要保存全部历史数据，故其压缩格式应选择**压缩比较高的，此处选择gzip**
    3. ODS层表名的命名规范为：**ods_表名_单分区增量全量标识**（inc/full）

2. log表建表 vim [create_table_log.sql](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/create_table_log.sql)
3. log表每日数据装载脚本 vim [hdfs_to_ods_log.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/hdfs_to_ods_log.sh)
4. 业务表建表 vim [create_table_ods.sql](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/create_table_ods.sql)
5. 业务表每日数据装载脚本 vim [hdfs_to_ods_db.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/hdfs_to_ods_db.sh)

## 数仓开发之DIM层(有难点)

1. DIM层设计要点：

    1. DIM层的设计依据是维度建模理论，该层存储维度模型的维度表。

    2. DIM层的数据存储格式为orc列式存储+snappy压缩。

    3. DIM层表名的命名规范为dim_表名_全量表或者拉链表标识（full/zip）

    4. **建表-> 确认字段 -> 确认首日数据装载和每日数据装载/拉链表设置 -> 确认需要的表（字段设置与沉淀）-> select语法 -> join选择-> 自动分区设置与否**

    5. 难点：

        * 每日装载和首日装载需要分别考虑。

        * **用户维度表（拉链表）**。

        * 日期维度表，需要每年的数据

2. 建表sql vim [create_table_dim.sql](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/create_table_dim.sql)
3. 首日装载脚本 vim [ods_to_dim_init.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/ods_to_dim_init.sh)
4. 每日装载脚本 vim [ods_to_dim.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/ods_to_dim.sh)

## 数仓开发之DWD层(有难点)

1. DWD层设计要点：

    1. DWD层的设计依据是维度建模理论，该层存储维度模型的事实表。

    2. DWD层的数据存储格式为orc列式存储+snappy压缩。

    3. DWD层表名的命名规范为dwd_数据域_表名_单分区增量全量标识（inc/full）

    4. **建表 -> 确认字段 -> 确认首日数据装载和每日数据装载/拉链表设置 -> 确认需要的表（字段设置与沉淀）-> select语法 -> join选择-> 自动分区设置与否**

    5. 难点：
        
        * 每日装载和首日装载需要分别考虑。

        * 多分区设定（需要设置分区模式）。
        
        * 退款退单的区别。流量表所属会话id的设置。

        * spark的不过滤STRUCT的bug。

        * 如何判断用户是否在这个会话第一次登陆。

        * 交易域购物车周期快照事实表需要对全量表进行统计，并且需要把 is_order 为1的记录去除。

        * 交易域加购事务事实表需要对增量表进行统计，并且需要每一个操作统计。

2. 建表语句 vim [create_table_dwd.sql](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/create_table_dwd.sql)

3. 首日装载脚本 [ods_to_dwd_init.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/ods_to_dwd_init.sh)

4. 每日装载脚本 [ods_to_dwd.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/ods_to_dwd.sh)


## 数仓开发之DWS层(有难点)

1. 设计要点：

    1. DWS层的设计参考指标体系。

    2. DWS层的数据存储格式为orc列式存储+snappy压缩。

    3. DWS层表名的命名规范为dws_数据域_统计粒度_业务过程_统计周期（1d/nd/td） 1d表示最近1日，nd表示最近n日，td表示历史至今。

    4. 难点：

        * dws是用来简化ads层的，如果ads层需要，可以随时加表。

        * 人数统计 去重问题：降低粒度，加字段，不能直接在dws层聚合

        * 表涉及的需求少的话，可以与其他业务过程表合并（相当于降低粒度）

        * 区分统计周期：1d，nd，td；td依赖1d，nd；nd依赖1d/粒度

2. 建表语句 vim [create_table_dws.sql](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/create_table_dws.sql)

3. 首日装载脚本 [dwd_to_dws_1d_init.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/dwd_to_dws_1d_init.sh)

4. 每日装载脚本 [dwd_to_dws_1d.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/dwd_to_dws_1d.sh)

5. 每日装载脚本 [dws_1d_to_dws_nd.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/dws_1d_to_dws_nd.sh)

6. 首日数据装载脚本 [dws_1d_to_dws_td_init.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/dws_1d_to_dws_td_init.sh)

7. 每日数据装载脚本 [dws_1d_to_dws_td.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/dws_1d_to_dws_td.sh)

## 数仓开发之ADS层

1. 根据各主题设定 统计

2. 难点

    * lateral view 之后 where 可以使用炸裂后的新字段过滤，使炸裂后新字段分割的数据不同。

    * union 去重时要保持两端数据一致

    * 桑基图 source 不能为空（lead函数将source为当前页面，而不是上个页面），不能存在环

    * 涉及时间变化（新增）的可以维护一个表在dws层

    * 用户变动统计：末次登陆日期

    * 用户留存率：当前日期前1，2，3，4，5，6，7天注册用户在今天还在的比例

3. 流量主题

|统计周期|统计粒度|指标|说明|
| :-- | :-- | :-- | :-- |
|最近1/7/30日|渠道|访客数|统计访问人数|
|最近1/7/30日|渠道|会话平均停留时长|统计每个会话平均停留时长|
|最近1/7/30日|渠道|会话平均浏览页面数|统计每个会话平均浏览页面数
|最近1/7/30日|渠道|会话总数|统计会话总数|
|最近1/7/30日|渠道|跳出率|只有一个页面的会话的比例|

4. 路径分析

<span style="text-align: center;display:block;">
<img src="./img/ADS层-路径分析.png"/></span>

5. 用户主题

|统计周期|指标|说明|
| :-- | :-- | :-- |
|最近1日|流失用户数|之前活跃过的用户，最近一段时间未活跃，就称为流失用户。此处要求统计7日前（只包含7日前当天）活跃，但最近7日未活跃的用户总数。|
|最近1日|回流用户数|之前的活跃用户，一段时间未活跃（流失），今日又活跃了，就称为回流用户。此处要求统计回流用户总数。|

6. 用户留存率

<span style="text-align: center;display:block;">
<img src="./img/ADS层-用户留存率.png"/></span>

7. 用户新增活跃统计

|统计周期|指标|指标说明|
| :-- | :-- | :-- |
|最近1、7、30日|新增用户数|略|
|最近1、7、30日|活跃用户数|略|

8. 用户行为漏斗分析。统计一个完整的购物流程各个阶段的人数，具体说明如下：

|统计周期|指标|说明|
| :-- | :-- | :-- |
|最近1、7、30日|首页浏览人数|略|
|最近1、7、30日|商品详情页浏览人数|略|
|最近1、7、30日|加购人数|略|
|最近1、7、30日|下单人数|略|
|最近1、7、30日|支付人数|支付成功人数|

9. 新增交易用户统计

|统计周期|指标|说明|
| :-- | :-- | :-- |
|最近1、7、30日|新增下单人数|略|
|最近1、7、30日|新增支付人数|略|

10. 商品主题

|统计周期|统计粒度|指标|说明|
| :-- | :-- | :-- | :-- |
|最近7、30日|品牌|复购率|重复购买人数占购买人数比例|
|最近1、7、30日|品牌|订单数|略|
|最近1、7、30日|品牌|订单人数||略|
|最近1、7、30日|品牌|退单数|略|
|最近1、7、30日|品牌|退单人数|略|
|最近1、7、30日|品类|订单数|略|
|最近1、7、30日|品类|订单人数|略|
|最近1、7、30日|品类|退单数|略|
|最近1、7、30日|品类|退单人数|略|
|||各分类商品购物车存量Top10|略|

11. 交易主题

|统计周期|指标|说明|
| :-- | :-- | :-- |
|最近1、7、30日|订单总额|订单最终金额|
|最近1、7、30日|订单数|略|
|最近1、7、30日|订单人数|略|
|最近1、7、30日|退单数|略|
|最近1、7、30日|退单人数|略|

|统计周期|统计粒度|指标|说明|
| :-- | :-- | :-- | :-- |
|最近1、7、30日|省份|订单数|略|
|最近1、7、30日|省份|订单金额|略|

12. 优惠券主题

|统计粒度|指标|说明|
| :-- | :-- | :-- |
|优惠券|补贴率|用券的订单明细优惠券减免金额总和/原始金额总和|

13. 活动主题

|统计粒度|指标|说明|
| :-- | :-- | :-- |
|活动|补贴率|参与促销活动的订单明细活动减免金额总和/原始金额总和|

14. 每日数据装载脚本 vim [dws_to_ads.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/dws_to_ads.sh)
15. 建表脚本 vim [create_table_ads.sql](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/create_table_ads.sql)

## 报表数据导出

1. 创建数据库

```sql
CREATE DATABASE IF NOT EXISTS gmall_report DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
```

2. 创建表脚本 vim [create_table_ads_mysql.sql](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/create_table_ads_mysql.sql)

3. 数据导出datax配置脚本 [gen_export_config.py](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/gen_export_config.py)脚本

4. 数据导出[gen_export_config.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/gen_export_config.sh)脚本

5. 编写每日导出脚本 vim [hdfs_to_mysql.sh](https://github.com/Ciaran-crop/conf/blob/master/DWConfiguration/hdfs_to_mysql.sh)

## 工作流调度

1. Apache DolphinScheduler是一个分布式、易扩展的可视化DAG工作流任务调度平台

2. DolphinScheduler核心架构

<span style="text-align: center;display:block;">
<img src="./img/DolphinScheduler核心架构.png"/></span>

3. DolphinScheduler的主要角色如下：

    * MasterServer采用分布式无中心设计理念，MasterServer主要负责 DAG 任务切分、任务提交、任务监控，并同时监听其它MasterServer和WorkerServer的健康状态。
    * WorkerServer也采用分布式无中心设计理念，WorkerServer主要负责任务的执行和提供日志服务。
    * ZooKeeper服务，系统中的MasterServer和WorkerServer节点都通过ZooKeeper来进行集群管理和容错。
    * Alert服务，提供告警相关服务。
    * API接口层，主要负责处理前端UI层的请求。
    * UI，系统的前端页面，提供系统的各种可视化操作界面。

4. DolphinScheduler部署(系统要求高)

    1. 单机模式(无需配置)。所有服务均集中于一个StandaloneServer进程中，并且其中内置了注册中心Zookeeper和数据库H2(**<16GB使用单机模式**) `bin/dolphinscheduler-daemon.sh start standalone-server`

    2. 伪集群模式。单台机器部署 DolphinScheduler 各项服务

    3. 集群模式

5. 安装

    1. 三台节点均需安装进程管理工具包psmisc `yum install -y psmisc`

    2. 解压 apache-dolphinscheduler-1.3.9-bin.tar.gz 

    3. 初始化数据库

    ```sql
    CREATE DATABASE dolphinscheduler DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
    CREATE USER 'dolphinscheduler'@'%' IDENTIFIED BY 'dolphinscheduler';
    ```

    4. 修改数据源配置文件

        1. 修改conf目录下的datasource.properties文件

        ```
        spring.datasource.driver-class-name=com.mysql.jdbc.Driver
        spring.datasource.url=jdbc:mysql://hadoop102:3306/dolphinscheduler?useUnicode=true&characterEncoding=UTF-8
        spring.datasource.username=dolphinscheduler
        spring.datasource.password=dolphinscheduler
        ```

        2. 拷贝MySQL驱动到DolphinScheduler的解压目录下的lib中

    5. 执行数据库初始化脚本 `script/create-dolphinscheduler.sh`

    6. 配置一键部署脚本。修改解压目录下的conf/config目录下的install_config.conf文件

    ```
    # postgresql or mysql
    dbtype="mysql"

    # db config
    # db address and port
    dbhost="hadoop102:3306"

    # db username
    username="dolphinscheduler"

    # database name
    dbname="dolphinscheduler"

    # db passwprd
    # NOTICE: if there are special characters, please use the \ to escape, for example, `[` escape to `\[`
    password="dolphinscheduler"

    # zk cluster
    zkQuorum="hadoop102:2181,hadoop103:2181,hadoop104:2181"

    # Note: the target installation path for dolphinscheduler, please not config as the same as the current path (pwd)
    installPath="/opt/module/dolphinscheduler"

    # deployment user
    # Note: the deployment user needs to have sudo privileges and permissions to operate hdfs. If hdfs is enabled, the root directory needs to be created by itself
    deployUser="ciaran"

    # resource storage type: HDFS, S3, NONE
    resourceStorageType="HDFS"

    # resource store on HDFS/S3 path, resource file will store to this hadoop hdfs path, self configuration, please make sure the directory exists on hdfs and have read write permissions. "/dolphinscheduler" is recommended
    resourceUploadPath="/dolphinscheduler"

    # if resourceStorageType is HDFS，defaultFS write namenode address，HA you need to put core-site.xml and hdfs-site.xml in the conf directory.
    # if S3，write S3 address，HA，for example ：s3a://dolphinscheduler，
    # Note，s3 be sure to create the root directory /dolphinscheduler
    defaultFS="hdfs://hadoop102:8020"

    # resourcemanager port, the default value is 8088 if not specified
    resourceManagerHttpAddressPort="8088"

    # if resourcemanager HA is enabled, please set the HA IPs; if resourcemanager is single, keep this value empty
    yarnHaIps=

    # if resourcemanager HA is enabled or not use resourcemanager, please keep the default value; If resourcemanager is single, you only need to replace ds1 to actual resourcemanager hostname
    singleYarnIp="hadoop103"

    # who have permissions to create directory under HDFS/S3 root path
    # Note: if kerberos is enabled, please config hdfsRootUser=
    hdfsRootUser="ciaran"

    # api server port
    apiServerPort="12345"


    # install hosts
    # Note: install the scheduled hostname list. If it is pseudo-distributed, just write a pseudo-distributed hostname
    ips="hadoop102,hadoop103,hadoop104"

    # ssh port, default 22
    # Note: if ssh port is not default, modify here
    sshPort="22"

    # run master machine
    # Note: list of hosts hostname for deploying master
    masters="hadoop102"

    # run worker machine
    # note: need to write the worker group name of each worker, the default value is "default"
    workers="hadoop102:default,hadoop103:default,hadoop104:default"

    # run alert machine
    # note: list of machine hostnames for deploying alert server
    alertServer="hadoop102"

    # run api machine
    # note: list of machine hostnames for deploying api server
    apiServers="hadoop102"
    ```

    7. 一键部署DolphinScheduler

        1. 启动zookeeper

        2. bin/install.sh

        3. 访问DolphinScheduler UI http://hadoop102:12345/dolphinscheduler 初始用户的用户名为：admin，密码为dolphinscheduler123
    
    8. DolphinScheduler启停命令

        1. 一键启停所有服务 `./bin/start-all.sh ./bin/stop-all.sh`

        2. 启停 Master`./bin/dolphinscheduler-daemon.sh start master-server ./bin/dolphinscheduler-daemon.sh stop master-server `

        3. 启停 Worker `./bin/dolphinscheduler-daemon.sh start worker-server ./bin/dolphinscheduler-daemon.sh stop worker-server`

        4. 启停 Api `./bin/dolphinscheduler-daemon.sh start api-server ./bin/dolphinscheduler-daemon.sh stop api-server`

        5. 启停 Logger `./bin/dolphinscheduler-daemon.sh start logger-server ./bin/dolphinscheduler-daemon.sh stop logger-server`

        6. 启停 Alert `./bin/dolphinscheduler-daemon.sh start alert-server ./bin/dolphinscheduler-daemon.sh stop alert-server`

6. DolphinScheduler入门

    1. 安全中心主要有租户管理、用户管理、告警组管理、Worker分组管理、队列管理、令牌管理等功能

    2. 队列管理: 此处的队列对应的是**Yarn调度器的资源队列**。故队列概念只对**跑在Yarn上的任务类型有效**。此处创建出的队列，**可供后续任务进行选择**。需要注意的是，在DolphinScheduler中创建队列，**并不会影响到Yarn调度器的队列配置**。

    3. 租户管理: 租户对应的是Linux系统用户，是Worker执行任务使用的用户。如果Worker所在节点没有这个用户，Worker会在执行任务时创建这个用户。**租户编码**：对应Worker执行任务所使用的用户名

    4. 用户管理: 用户对应的是DolphinScheduler的用户，用于登录DolphinScheduler。用户分管理员用户和普通用户。默认情况下，管理员只有授权和用户管理等权限，而普通用户只有创建项目，定义工作流、执行工作流等权限。

    5. 告警组管理: 告警组可包含多名用户，用于指定告警发送对象。

    6. Worker分组管理: 在任务执行时，可以将任务分配给指定Worker组，最终由该组中的Worker节点执行该任务。默认情况下，所有Worker均位于default组。

    7. 令牌管理: 令牌用于通过接口访问DolphinScheduler各项服务时的用户验证。普通用户通过UI页面访问各项服务时，无需使用令牌。若需将DolphinScheduler与第三方服务进行集成，则需调用其接口，此时需使用令牌。

    8. 项目管理: 创建项目

    9. 工作流基础配置(工作流定义,工作流实例，任务实例)

        * 工作流定义：用于定义工作流，包括工作流各节点任务详情及各节点依赖关系等。
        * 工作流实例：工作流每执行一次就会生成一个工作流示例。此处可查看正在运行的工作流以及已经完成的工作流。
        * 任务实例：工作流中的一个节点任务，每执行一次就会生成一个任务实例。此处可用于查看正在执行的节点任务以及已经完成的节点任务。

    10. 提交执行工作流

        1. 提交执行工作流

        2. 单次运行工作流

        3. 定时执行工作流/点击定时/配置定时规则/定时管理/定时上线
    


7. DolphinScheduler进阶

    1. 工作流传参。任务节点可通过${参数名}引用参数

    2. 局部参数在工作流Node-A节点下

    3. 全局参数在保存工作流时设定

    4. 系统内置参数。如 ${system.biz.date}

    5. 时间自定义参数。$[自己设定格式]，如$[yyyyMMdd]

    6. 引用依赖资源/在资源中心创建脚本/在节点资源处选择资源

    7. 告警通知

        1. 准备电子邮箱账户/开启SMTP服务

        2. 配置AlertServer /opt/module/dolphinscheduler/conf/alert.properties(如果使用云服务器，由于25号端口不能开启，需要使用加密协议，使用其他端口发送邮件)

        ```
        # 不使用加密协议
        #alert type is EMAIL/SMS
        alert.type=EMAIL

        # mail server configuration
        mail.protocol=SMTP
        mail.server.host=smtp.qq.com
        mail.server.port=25
        mail.sender=*********@qq.com
        mail.user=*********@qq.com
        mail.passwd=*************
        # TLS
        mail.smtp.starttls.enable=false
        # SSL
        mail.smtp.ssl.enable=false
        mail.smtp.ssl.trust=smtp.exmail.qq.com
        ```

        ```
        # 使用SSL加密协议
        #alert type is EMAIL/SMS
        alert.type=EMAIL

        # mail server configuration
        mail.protocol=SMTP
        mail.server.host=smtp.qq.com
        mail.server.port=465
        mail.sender=*********@qq.com
        mail.user=*********@qq.com
        mail.passwd=*************
        # TLS
        mail.smtp.starttls.enable=false
        # SSL
        mail.smtp.ssl.enable=true
        mail.smtp.ssl.trust=smtp.qq.com
        ```

    
    8. 工作流失败重跑:/工作流实例中有两个按钮叫做重跑(从起点开始)和恢复失败(从断点开始)

8. 工作流调度实操

    1. 向DolphinScheduler资源中心上传工作流所需脚本/创建文件夹/上传工作流所需脚本

    2. 向DolphinScheduler的WorkerServer节点分发脚本依赖的组件(单机模式不需要)

    3. 修改DolphinScheduler环境变量配置文件并分发 vim /opt/module/dolphinscheduler/conf/env/dolphinscheduler_env.sh 

    ```shell
    export HADOOP_HOME=/opt/module/hadoop-3.1.3
    export HADOOP_CONF_DIR=/opt/module/hadoop-3.1.3/etc/hadoop
    export SPARK_HOME=/opt/module/spark
    # SPARK_HOME 可以不需要
    export SPARK_HOME2=/opt/soft/spark2
    export PYTHON_HOME=/opt/soft/python
    export JAVA_HOME=/opt/module/jdk1.8.0_212
    export HIVE_HOME=/opt/module/hive
    export FLINK_HOME=/opt/soft/flink
    export DATAX_HOME=/opt/module/datax

    export PATH=$HADOOP_HOME/bin:$SPARK_HOME1/bin:$SPARK_HOME2/bin:$PYTHON_HOME:$JAVA_HOME/bin:$HIVE_HOME/bin:$FLINK_HOME/bin:$DATAX_HOME/bin:$PATH 
    ```

    4. 创建工作流(需要先创建工作流)，设定节点，配置后保存工作流，设定dt变量

    5. 上线工作流/执行工作流

## 数据可视化Superset

1. 现代的数据探索和可视化平台

2. Superset官网地址：http://superset.apache.org/

3. 安装Python环境。要求Python3.7的环境

4. 安装 miniconda  https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh `bash Miniconda3-latest-Linux-x86_64.sh`
 
5. 加载环境变量 `source ~/.bashrc` 取消激活base环境 `conda config --set auto_activate_base false`

6. 配置conda国内镜像

```shell
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
conda config --set show_channel_urls yes
```

7. 创建Python3.7环境 `conda create --name superset python=3.7`激活superset环境 `conda activate superset`退出环境 `conda deactivate`

8. 部署 Superset 

    1. 安装依赖 `yum install -y gcc gcc-c++ libffi-devel python-devel python-pip python-wheel python-setuptools openssl-devel cyrus-sasl-devel openldap-devel`

    2. 安装 Superset 

    ```shell
    pip install --upgrade setuptools pip -i https://pypi.douban.com/simple/
    pip install apache-superset -i https://pypi.douban.com/simple/
    ```

    3. 初始化数据库 `superset db upgrade`

    4. 创建 管理员用户 。flask是一个python web框架，Superset使用的就是flask，需要指定FLASK_APP为superset

    ```shell
    export FLASK_APP=superset
    superset fab create-admin
    ```

    5. Superset初始化 `superset init`

9. 启动Supterset

    1. 安装gunicorn。gunicorn是一个Python Web Server，可以和java中的TomCat类比

    2. gunicorn启动superset服务：`gunicorn --workers 5 --timeout 120 --bind hadoop102:8787  "superset.app:create_app()" --daemon`

    3. 登录Superset http://hadoop102:8787

10. 停止superset `ps -ef | awk '/superset/ && !/awk/{print $2}' | xargs kill -9`

11. superset启停脚本  vim superset.sh

```shell
#!/bin/bash

superset_status(){
    result=`ps -ef | awk '/gunicorn/ && !/awk/{print $2}' | wc -l`
    if [[ $result -eq 0 ]]; then
        return 0
    else
        return 1
    fi
}
superset_start(){
        source ~/.bashrc
        superset_status >/dev/null 2>&1
        if [[ $? -eq 0 ]]; then
            conda activate superset ; gunicorn --workers 5 --timeout 120 --bind hadoop102:8787 --daemon 'superset.app:create_app()'
        else
            echo "superset正在运行"
        fi

}

superset_stop(){
    superset_status >/dev/null 2>&1
    if [[ $? -eq 0 ]]; then
        echo "superset未在运行"
    else
        ps -ef | awk '/gunicorn/ && !/awk/{print $2}' | xargs kill -9
    fi
}


case $1 in
    start )
        echo "启动Superset"
        superset_start
    ;;
    stop )
        echo "停止Superset"
        superset_stop
    ;;
    restart )
        echo "重启Superset"
        superset_stop
        superset_start
    ;;
    status )
        superset_status >/dev/null 2>&1
        if [[ $? -eq 0 ]]; then
            echo "superset未在运行"
        else
            echo "superset正在运行"
        fi
esac
```

12. Superset使用

    0. 对接不同的数据源，需安装不同的依赖，以下地址为官网说明 https://superset.apache.org/docs/databases/installing-database-drivers

    1. 对接MySQL数据源。安装依赖 `conda install mysqlclient`。重启Superset
    
    2. 数据源配置。Database配置/"mysql://root:000000@hadoop102:3306/gmall_report?charset=utf8"/Datasetse配置/添加dataset（相当于table）

    3. 制作仪表盘：/dashboards/编辑/可调整仪表盘自动刷新时间

    4. 创建图标：/charts/选择数据源，表类型/创建后配置表/Run Query/命名或其它操作/保存（可以保存到仪表盘中）

    5. 字段说明

    |||
    | :-- | :-- |
    |time column| 时期字段，会自动筛选出类型为date或datetime类型，自动group by 这个字段|
    |time grain| 时间粒度，选择一天或者一月等，做为x轴间距|
    |time range| 时间范围|
    |metric| 展示指标，y轴，可聚合|
    |chmparison| period lag 比较延迟，1表示与前1日相比|

    6. 其他表自行搜索


# 补充(有重点)

## Hue

是个好东西，可以整合很多框架，提高用户体验，可视化操作。

## DWT层

对DWS层做一层汇总，变成某一主题的宽表（多字段，很多）

## 即席查询(有重点)

1. **理解**：即席查询通常查询DWD，DWS，DIM层数据，经常发生在查询ADS层没有的结果的时候，这时hive查询太慢，通过Presto，Kylin等组件，引擎，实现快速查询，即查即用。这些组件也有图形化，也有接口，与其他组件协同工作。实现快速，效率，便捷的查询。

### Presto

1. 是一个开源的分布式SQL**查询引擎**，数据量支持GB到PB，用于秒级查询。

2. 不是数据库，不能处理**在线事务**(OLTP)

3. 架构

    <span style="text-align: center;display:block;">
    <img src="./img/Presto架构.png"/></span>

    1. 由一个Coordinator和多个Worker组成

    2. 流程

        1. 由客户端提交查询，从Presto命令行CLI提交到Coordinator。

        2. Coordinator解析查询计划，然后把任务分发给Worker执行。
        3. Worker负责执行任务和处理数据
        4. Catolog表示数据源。一个Catelog包含Schema和Connector
        5. Connector是适配器，用于Presto和数据源（如Hive、Redis）的连接，类似于JDBC
        6. Schema类似于Mysql中数据库，Table类似于MySQL中表
        7. Coordinator是负责从Worker获取结果并返回最终结果给Client

4. 优点

    1. 基于内存，减少磁盘IO，计算更快

    2. 连接多个数据源，跨数据源
5. 缺点 

    1. 连表查，临时数据大，速度变慢。如果数据大，会边读变算，再清内存，不会一次性读入。

6. [Presto、Impala性能比较](https://blog.csdn.net/u012551524/article/details/79124532)

    1. Impala性能稍领先于Presto，但是Presto在数据源支持上非常丰富，包括Hive、图数据库、传统关系型数据库、Redis等

#### Presto安装

##### Presto Server安装

1. [官网地址](https://prestodb.github.io/)
2. [下载地址](https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.196/presto-server-0.196.tar.gz)
3. 将tar.gz文件导入服务器的/opt/software目录下，并解压到/opt/module目录
4. 修改文件夹名称为presto
5. 进入到/opt/module/presto目录，并创建存储数据文件夹 `mkdir data`
6. 进入到/opt/module/presto目录，并创建存储配置文件文件夹`mkdir etc`
7. 配置在/opt/module/presto/etc目录下添加jvm.config配置文件
`vim jvm.config`添加如下内容

```
-server
-Xmx16G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
```

8. Presto可以支持多个数据源，在Presto里面叫catalog，这里我们配置支持Hive的数据源(**其他数据源同理**)，配置一个Hive的catalog `mkdir catalog` && `vim hive.properties` 添加如下内容
```
connector.name=hive-hadoop2
hive.metastore.uri=thrift://hadoop102:9083
```
9. 将服务器上的presto分发到其他服务器
10. 分发之后，分别进入hadoop102、hadoop103、hadoop104三台主机的/opt/module/presto/etc的路径。配置node属性，node id每个节点都不一样。
`vim node.properties`
```
node.environment=production
node.id=ffffffff-ffff-ffff-ffff-ffffffffffff
node.data-dir=/opt/module/presto/data
```
`vim node.properties`
```
node.environment=production
node.id=ffffffff-ffff-ffff-ffff-fffffffffffe
node.data-dir=/opt/module/presto/data
```
`vim node.properties`
```
node.environment=production
node.id=ffffffff-ffff-ffff-ffff-fffffffffffd
node.data-dir=/opt/module/presto/data
```
11. Presto是由一个coordinator节点和多个worker节点组成。在hadoop102上配置成coordinator，在hadoop103、hadoop104上配置为worker。
12. hadoop102上配置coordinator节点 `vim config.properties` 添加内容如下
```
coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=8881
query.max-memory=50GB
discovery-server.enabled=true
discovery.uri=http://hadoop102:8881
```
13. hadoop103、hadoop104上配置worker节点 `vim config.properties` 添加内容如下
```
coordinator=false
http-server.http.port=8881
query.max-memory=50GB
discovery.uri=http://hadoop102:8881
```
14. 在hadoop102的/opt/module/hive目录下，启动Hive Metastore ```nohup bin/hive --service metastore >/dev/null 2>&1 &```
15. 分别在hadoop102、hadoop103、hadoop104上启动Presto Server
    1. 前台启动Presto，控制台显示日志 `bin/launcher run`
    2. 后台启动Presto `bin/launcher start`
16. 日志查看路径/opt/module/presto/data/var/log



##### Presto命令行Client安装

1. [下载Presto的客户端](https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.196/presto-cli-0.196-executable.jar)
2. 将jar文件上传到服务器的/opt/module/presto文件夹下
3. 修改文件名称 为prestocli 
4. 增加执行权限
5. 启动prestocli `./prestocli --server hadoop102:8881 --catalog hive --schema default`
6. Presto命令行操作
7. Presto的命令行操作，相当于Hive命令行操作。每个表必须要加上schema。 `select * from schema.table limit 100`

##### Presto可视化Client安装

1. 将zip文件上传到服务器的/opt/module目录
2. 解压缩 unzip
3. 进入到/opt/module/yanagishima-18.0/conf文件夹，编写yanagishima.properties配置 `vim yanagishima.properties` 添加如下内容
```
jetty.port=7080
presto.datasources=atguigu-presto
presto.coordinator.server.atguigu-presto=http://hadoop102:8881
catalog.atguigu-presto=hive
schema.atguigu-presto=default
sql.query.engines=presto
```
4. 在/opt/module/yanagishima-18.0路径下启动yanagishima `nohup bin/yanagishima-start.sh >y.log 2>&1 &`
5. 启动web页面 http://hadoop102:7080 
6. 查看表结构。这里有个Tree View，可以查看所有表的结构，包括Schema、表、字段等。

#### Presto优化之数据存储

1. 合理设置分区

与Hive类似，Presto会根据元数据信息**读取分区数据**，合理的分区能减少Presto数据读取量，提升查询性能。

2. 使用列式存储

Presto对**ORC文件读取做了特定优化**，因此在Hive中创建Presto使用的表时，建议采用ORC格式存储。相对于Parquet，Presto对ORC支持更好。

3. 使用压缩

数据压缩可以减少节点间数据传输对IO带宽压力，对于即席查询需要快速解压，**建议采用Snappy压缩**。

#### Presto优化之查询SQL

1. 只选择使用的字段

由于采用列式存储，选择需要的字段可加快字段的读取、减少数据量。避免采用*读取所有字段。

```sql
[GOOD]: SELECT time, user, host FROM tbl
[BAD]:  SELECT * FROM tbl
```

2. 过滤条件必须加上分区字段

对于有分区的表，where语句中优先使用分区字段进行过滤。acct_day是分区字段，visit_time是具体访问时间。

```sql
[GOOD]: SELECT time, user, host FROM tbl where acct_day=20171101
[BAD]:  SELECT * FROM tbl where visit_time=20171101
```

3. Group By语句优化

合理安排Group by语句中字段顺序对性能有一定提升。将Group By语句中字段按照每个字段distinct数据多少进行降序排列。

```sql
[GOOD]: SELECT GROUP BY uid, gender
[BAD]:  SELECT GROUP BY gender, uid
```

4. Order by时使用Limit

Order by需要扫描数据到单个worker节点进行排序，导致单个worker需要大量内存。如果是查询Top N或者Bottom N，使用limit可减少排序计算和内存压力。

```sql
[GOOD]: SELECT * FROM tbl ORDER BY time LIMIT 100
[BAD]:  SELECT * FROM tbl ORDER BY time
```

5. 使用Join语句时将大表放在左边

Presto中join的默认算法是broadcast join，即将join左边的表分割到多个worker，然后将join右边的表数据整个复制一份发送到每个worker进行计算。如果右边的表数据量太大，则可能会报内存溢出错误。

```sql
[GOOD] SELECT ... FROM large_table l join small_table s on l.id = s.id
[BAD] SELECT ... FROM small_table s join large_table l on l.id = s.id
```

####  注意事项

1. 字段名引用

避免和关键字冲突：MySQL对字段加反引号`、Presto对字段加双引号分割当然，如果字段名称不是关键字，可以不加这个双引号。

2. 时间函数

对于Timestamp，需要进行比较的时候，需要添加Timestamp关键字，而MySQL中对Timestamp可以直接进行比较。

```sql
/*MySQL的写法*/
SELECT t FROM a WHERE t > '2017-01-01 00:00:00'; 
/*Presto中的写法*/
SELECT t FROM a WHERE t > timestamp '2017-01-01 00:00:00';
```

3. 不支持INSERT OVERWRITE语法

Presto中不支持insert overwrite语法，只能先delete，然后insert into。

4. PARQUET格式

Presto目前支持Parquet格式，支持查询，但不支持insert。

### Kylin

1. 开源的分布式**分析引擎** Hadoop/Spark之上的SQL查询接口及**多维分析**（OLAP）能力以**支持超大规模数据**

2. 架构

    <span style="text-align: center;display:block;">
    <img src="./img/Kylin架构.png"/></span>

    1. REST Server

    REST Server是一套面向应用程序开发的**入口点**，旨在实现针对Kylin平台的应用开发工作。 此类应用程序可以提供查询、获取结果、触发cube构建任务、获取元数据以及获取用户权限等等。另外可以通过Restful接口**实现SQL查询**。

    2. 查询引擎（Query Engine）
    
    当cube准备就绪后，查询引擎就能够**获取并解析用户查询**。它随后会与系统中的其它组件**进行交互**，从而向用户返回对应的结果。 

    3. 路由器（Routing）

    在最初设计时曾考虑过将Kylin不能执行的查询**引导去Hive中继续执行**，但在实践后发现Hive与Kylin的速度差异过大，导致用户无法对查询的速度有一致的期望，很可能大多数查询几秒内就返回结果了，而有些查询则要等几分钟到几十分钟，因此体验非常糟糕。最后这个路由功能在发行版中**默认关闭**。

    4. 元数据管理工具（Metadata）

    Kylin是一款元数据驱动型应用程序。元数据管理工具是一大关键性组件，用于对保存在Kylin当中的所有元数据进行管理，其中包括最为重要的cube元数据。其它全部组件的正常运作都需以元数据管理工具为基础。 Kylin的元数据存储在hbase中。 

    5. 任务引擎（Cube Build Engine）

    这套引擎的设计目的在于**处理所有离线任务**，其中包括shell脚本、Java API以及Map Reduce任务等等。任务引擎对Kylin当中的全部任务加以管理与协调，从而确保每一项任务都能得到切实执行并解决其间出现的故障。

3. Kylin特点

    1. Kylin的主要特点包括支持SQL接口、支持超大规模数据集、亚秒级响应、可伸缩性、高吞吐率、BI工具集成等。

    2. 标准SQL接口：Kylin是以标准的SQL作为对外服务的接口。

    3. 支持超大数据集：Kylin对于大数据的支撑能力可能是目前所有技术中最为领先的。早在2015年eBay的生产环境中就能支持百亿记录的秒级查询，之后在移动的应用场景中又有了千亿记录秒级查询的案例。

    4. 亚秒级响应：Kylin拥有优异的查询相应速度，这点得益于预计算，很多复杂的计算，比如连接、聚合，在离线的预计算过程中就已经完成，这大大降低了查询时刻所需的计算量，提高了响应速度。

    5. 可伸缩性和高吞吐率：单节点Kylin可实现每秒70个查询，还可以搭建Kylin的集群。

    6. BI工具集成

        1. Kylin可以与现有的BI工具集成，具体包括如下内容。

        2. ODBC：与Tableau、Excel、PowerBI等工具集成

        3. JDBC：与Saiku、BIRT等Java工具集成

        5. RestAPI：与JavaScript、Web网页集成

        5. Kylin开发团队还贡献了Zepplin的插件，也可以使用Zepplin来访问Kylin服务。

#### Kylin安装

1. Kylin依赖环境: Hadoop、Hive、Zookeeper、HBase,及其环境变量

2. 上传Kylin安装包apache-kylin-3.0.2-bin.tar.gz

3. 解压apache-kylin-3.0.2-bin.tar.gz到/opt/module并改名 为 kylin

4.  Kylin兼容性问题。修改`/opt/module/kylin/bin/find-spark-dependency.sh`，输入排除冲突的jar包的名字。

    例如：jackson和metastore jar包冲突：

    需要增加的内容：! -name '*jackson*' ! -name '*metastore*'
    注意前后保留空格

    <span style="text-align: center;display:block;"><img src="./img/Kylin兼容性问题.png"/></span>

5. 启动

    1. 启动Kylin之前，需先启动Hadoop（hdfs，yarn，jobhistoryserver）、Zookeeper、Hbase

    2. 启动Kylin `bin/kylin.sh start`

    3. 在 http://hadoop102:7070/kylin 查看Web页面

    4. 用户名为：ADMIN，密码为：KYLIN

6. 关闭Kylin `bin/kylin.sh stop`


#### Kylin使用

1. **注意事项**：Kylin不能处理Hive表中的复杂数据类型（Array,Map,Struct）

2. 解决方案：将包含复杂数据类型的表中的复杂数据类型**分散成多个字段**。

3. 流程(具体不展开)

    1. 创建工程

    2. 获取数据源

    3. 创建model

    4. 构建cube

4. 使用进阶

    1. 每日全量维度表及拉链维度表重复Key问题如何处理：对数据过滤，**只选取最新数据**。
    
#### Kylin Cube构建原理(难点)

1. 维度：即观察数据的角度。

2. 维度：即观察数据的角度。

3. 对于N个维度来说，组合的所有可能性共有2^n种

4. 将度量值做**聚合计算**，然后将结果保存为一个**物化视图，称为Cuboid**。**所有维度组合的Cuboid**作为一个整体，称为**Cube**。也就是说对所有维度的所有度量值做聚合计算得到的一些结果作为一个整体，称为Cube。Cube包含很多计算结果。

5. **逐层算法**

    1. 在逐层算法中，按维度数逐层减少来计算，每个层级的计算（除了第一层，它是从原始数据聚合而来），是基于它上一层级的结果来计算的。比如，**[Group by A, B]的结果，可以基于[Group by A, B, C]的结果，通过去掉C后聚合得来的**；这样可以减少重复计算；当 0维度Cuboid计算出来的时候，整个Cube的计算也就完成了。

    2. 每一轮的计算都是一个MapReduce任务，且串行执行；一个N维的Cube，至少需要N次MapReduce Job。

    3. 算法优点：

        1. 此算法充分利用了MapReduce的优点，处理了中间复杂的排序和shuffle工作，故而算法代码清晰简单，易于维护；

        2. 受益于Hadoop的日趋成熟，此**算法非常稳定**，即便是集群资源紧张时，也能保证最终能够完成。

    4. 算法缺点：

        1. 当Cube有比较多维度的时候，所需要的MapReduce任务也相应增加；由于Hadoop的任务调度**需要耗费额外资源**，特别是集群较庞大的时候，反复递交任务造成的额外开销会相当可观；

        2. 由于Mapper逻辑中并未进行聚合操作，所以每轮MR的**shuffle工作量都很大**，导致**效率低下**。

        3. 对HDFS的**读写操作较多**：由于每一层计算的输出会用做下一层计算的输入，这些Key-Value需要写到HDFS上；当所有计算都完成后，Kylin还需要额外的一轮任务将这些文件转成HBase的HFile格式，以导入到HBase中去；

    5. 总体而言，该算法的**效率较低，但是稳定**，尤其是当Cube维度数较大的时候。


6. **快速构建算法（inmem）**

    1. 也被称作“逐段”(By Segment) 或“逐块”(By Split) 算法，从1.5.x开始引入该算法，该算法的**主要思想是**：每个Mapper将其所分配到的数据块，计算成一个完整的小Cube 段（**包含所有Cuboid**）。每个Mapper将计算完的Cube段输出给Reducer做合并，生成大Cube，也就是最终结果。

    2. 类似**分区**计算

    3.  Mapper会利用**内存做预聚合**，算出所有组合；Mapper输出的每个Key都是不同的，这样会减少输出到Hadoop MapReduce的数据量，Combiner也不再需要；
    
    4. 一轮MapReduce便会完成所有层次的计算，**减少Hadoop任务的调配**。

7. **Cube存储原理**

    1. 维度字典表：存储维度对应的值

    <span style="text-align: center;display:block;">
    <img src="./img/KylinCube存储原理-维度字典表.png"/></span>

    2. Hbase K-V：通过维度字段对应的值，通过**Cuboid id和维度字段对应的值组合**形成Hbase中的**RowKey**，查询到value

    <span style="text-align: center;display:block;">
    <img src="./img/KylinCube存储原理-HbaseK-Vpng.png"/></span>


#### Kylin Cube构建优化

1. 使用衍生维度（derived dimension）

<span style="text-align: center;display:block;">
<img src="./img/衍生维度.png"/></span>

2. 使用聚合组（Aggregation group）：聚合组（Aggregation Group）是一种强大的剪枝工具。三种可选的方式定义以下
        
    1. 强制维度（Mandatory），如果一个维度被定义为强制维度，那么这个分组产生的所有Cuboid中每一个Cuboid都会包含该维度。每个分组中都可以有0个、1个或多个强制维度。如下图

        <span style="text-align: center;display:block;">
        <img src="./img/Kylin-强制维度.png"/></span>
    
    2. 层级维度（Hierarchy），每个层级包含两个或更多个维度。如下图

        <span style="text-align: center;display:block;">
        <img src="./img/Kylin-层级维度.png"/></span>

    3. 联合维度（Joint），每个联合中包含两个或更多个维度，如果某些列形成一个联合，那么在该分组产生的任何Cuboid中，这些联合维度**要么一起出现，要么都不出现**

        <span style="text-align: center;display:block;">
        <img src="./img/Kylin-联合维度.png"/></span>

    4. 这些操作可以在Cube Designer的Advanced Setting中的Aggregation Groups区域完成

3. Row Key优化

    1. 被用作过滤的维度放在前边

    2. 基数大的维度放在基数小的维度前边

#### Kylin BI工具集成

1. JDBC

    1. 导入依赖

    ```xml
        <dependency>
            <groupId>org.apache.kylin</groupId>
            <artifactId>kylin-jdbc</artifactId>
            <version>3.0.2</version>
        </dependency>
    ```

    2. 编码

    ```java
    import java.sql.*;

    public class TestKylin {

        public static void main(String[] args) throws Exception {

            //Kylin_JDBC 驱动
            String KYLIN_DRIVER = "org.apache.kylin.jdbc.Driver";

            //Kylin_URL
            String KYLIN_URL = "jdbc:kylin://hadoop102:7070/FirstProject";

            //Kylin的用户名
            String KYLIN_USER = "ADMIN";

            //Kylin的密码
            String KYLIN_PASSWD = "KYLIN";

            //添加驱动信息
            Class.forName(KYLIN_DRIVER);

            //获取连接
            Connection connection = DriverManager.getConnection(KYLIN_URL, KYLIN_USER, KYLIN_PASSWD);

            //预编译SQL
            PreparedStatement ps = connection.prepareStatement("SELECT sum(sal) FROM emp group by deptno");

            //执行查询
            ResultSet resultSet = ps.executeQuery();

            //遍历打印
            while (resultSet.next()) {
                System.out.println(resultSet.getInt(1));
            }
        }
    }
    ```

2. Zepplin

    1. Zepplin安装与启动：上传 tgz，解压，修改名称，启动 `bin/zeppelin-daemon.sh start`，查看 http://hadoop102:8080

    2. 配置Zepplin支持Kylin：点击右上角anonymous选择Interpreter。搜索Kylin插件并修改相应的配置（url等）。修改完成点击Save完成

    3. 操作：点击Notebook创建新的note；编写查询；执行查询；显示结果。其他图表格式


## 集群监控(有重点)

1. Zabbix是一款能够监控各种网络参数以及服务器健康性和完整性的软件

2. 提供了出色的报告和数据可视化功能

3. 基础架构

<span style="text-align: center;display:block;"><img src="./img/Zabbix基础架构.png"/></span>

### Zabbix部署

1. agent：每个节点；server：一个节点；zabbix-web与server一个节点。

2. 关闭SELinux `/etc/selinux/config` 修改 `SELINUX=disabled`,重启机器

3. 安装zabbix的yum仓库(所有节点) `rpm -Uvh https://mirrors.aliyun.com/zabbix/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm`

4. 安装Software Collections仓库(所有节点) `yum install -y centos-release-scl`

5. 修改为阿里云镜像(sed语法) `sed -i 's/http:\/\/repo.zabbix.com/https:\/\/mirrors.aliyun.com\/zabbix/g' /etc/yum.repos.d/zabbix.repo`

6. 启用zabbix-web仓库。`/etc/yum.repos.d/zabbix.repo`文件，做如下修改: **[zabbix-frontend]下的enable改为1**

7. 在server端 `yum install -y zabbix-server-mysql zabbix-agent zabbix-web-mysql-scl zabbix-apache-conf-scl`

8. 在agent端  `yum install -y zabbix-agent`

9. 创建zabbix数据库

```sql
create database zabbix character set utf8 collate utf8_bin
```

10. 导入Zabbix建表语句 `zcat /usr/share/doc/zabbix-server-mysql-5.0.9/create.sql.gz | mysql -uroot -p密码 zabbix`

11. 配置Zabbix_Server。修改zabbix-server配置文件`vim /etc/zabbix/zabbix_server.conf`

```
DBHost=hadoop102
DBName=zabbix
DBUser=root
DBPassword=密码
```

12. 配置Zabbix_Agent（所有节点）`vim /etc/zabbix/zabbix_agentd.conf`

```
Server=hadoop102
#ServerActive=127.0.0.1
#Hostname=Zabbix server
```

13. 配置Zabbix_Web时区。`vim /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf`

```properties
[zabbix]
user = apache
group = apache

listen = /var/opt/rh/rh-php72/run/php-fpm/zabbix.sock
listen.acl_users = apache
listen.allowed_clients = 127.0.0.1

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35

php_value[session.save_handler] = files
php_value[session.save_path]    = /var/opt/rh/rh-php72/lib/php/session/

php_value[max_execution_time] = 300
php_value[memory_limit] = 128M
php_value[post_max_size] = 16M
php_value[upload_max_filesize] = 2M
php_value[max_input_time] = 300
php_value[max_input_vars] = 10000
php_value[date.timezone] = Asia/Shanghai
```

14. 启动停止Zabbix

```
[server端]systemctl start/stop/enable  zabbix-server zabbix-agent httpd rh-php72-php-fpm
[agent端]systemctl start/stop/enable zabbix-agent
```

15. 连接Zabbix_Web数据库。浏览器访问http://hadoop102/zabbix/。用户名：Admin，密码：zabbix

16. 点击User Settings，语言设成中文

### Zabbix使用

1. Zabbix术语

    * 主机（Host）: 一台你想监控的网络设备

    * 监控项（Item）: 你想要接收的主机的特定数据，一个度量数据

    * 触发器（Trigger）: 一个被用于定义问题阈值和“评估”监控项接收到的数据的逻辑表达式

    * 动作（Action）: 一个对事件做出反应的预定义的操作，比如邮件通知。

2. Zabbix实战

    1. 创建Host: 点击配置/主机/创建主机

    2. 创建监控项（Items）: 点击监控项（Items）,点击创建，配置，最后查看。

    3. 创建触发器（Trigger）：点击配置/主机/触发器，创建，编辑触发器。

    4. 创建动作（Action）：点击配置/动作/创建动作，编辑动作。

    5. 创建报警媒介类型（Media type）：需要开启邮箱SMTP服务，点击管理/报警媒介类型/Email，编辑Email。SMTP服务器（qq：smtp.qq.com）,测试Email, Email绑定收件人(User settings/报警媒介/添加/更新)

    6. 测试：关闭hdfs。查看邮件

3. 创建模板，将通用的监控项，触发器，动作组合，存在模板里，使用

    1. 点击配置/模板/创建模板，、
    2. 配置模板名称/群组
    3. 配置监控项，配置触发器，配置动作
    4. 应用模板
    5. 测试

### 集成 Grafana

1. 下载Grafana安装包,并上传

官方仓库：https://dl.grafana.com/oss/release/grafana-7.4.3-1.x86_64.rpm
国内镜像：https://repo.huaweicloud.com/grafana/7.4.3/grafana-7.4.3-1.x86_64.rpm

2. 使用rpm安装Grafana `rpm -ivh grafana-7.4.3-1.x86_64.rpm`

3. 启动Grafana `systemctl start grafana-server`

4. 访问Grafana页面; http://hadoop102:3000/; 首次登录用户名和密码均为 admin

5. 使用：创建Dashboard；新建panel，选择数据源，选择合适的可视化类型，保存Dashboard和panel，并命名

6. 集成Zabbix

    1. 配置数据源：Configuration/Data Sources/Add data source/选择所需数据源（如没有所需数据源，点击find more data...，获取更多数据源）/搜索zabbix，并点击搜索结果/按照所需插件的说明进行部署

    2. 部署步骤

        1. 安装插件 `grafana-cli plugins install alexanderzobnin-zabbix-app`

        2. 重启Grafana `systemctl restart grafana-server`
    
    3. 启用插件(第一步步骤)，配置url（http://host/zabbix/api_jsonrpc.php），basic auth
    为开，User，Password，Username，password为Admin和zabbix（或者自己改的密码）

7. 集成案例：点击Dashboards/新建panel/选择数据源/选择监控项/选择图标/保存

## 用户认证(复杂得要死)

1. Kerberos是一种计算机网络认证协议，用来在非安全网络中，对个人通信以安全的手段进行**身份认证**

2. 术语

    * KDC（Key Distribute Center）：密钥分发中心，负责存储用户信息，管理发放票据。
    * Realm：Kerberos所管理的一个领域或范围，称之为一个Realm。
    * Rrincipal：Kerberos所管理的一个用户或者一个服务，可以理解为Kerberos中保存的一个账号，其格式通常如下：primary/instance@realm
    * keytab：Kerberos中的用户认证，可通过密码或者密钥文件证明身份，keytab指密钥文件。

3. Kerberos认证原理

    <span style="text-align: center;display:block;"><img src="./img/Kerbero-认证原理.png"/></span>

    1. client 向hdfs发送请求
    2. client 向KDC认证 `kinit`
    3. KDC的AS，认证服务器向Databasse获取principal，发送TGT(Ticket granting ticket) 给 client
    4. client 收到TGT，向TGS服务器发送TGT
    5. TGS收到TGT，从database获取principal，验证TGT身份，成功就返回Server Ticket。
    6. client 获得Server Ticket，向hdfs发送请求，得到结果。

### Kerberos 部署

1. 选择集群中的一台主机（hadoop102）作为Kerberos服务端，安装KDC，所有主机都需要部署Kerberos客户端。

2. 服务端：`yum install -y krb5-server`

3. 客户端：`yum install -y krb5-workstation krb5-libs`

4. 修改配置文件

5. 服务端： `vim /var/kerberos/krb5kdc/kdc.conf`

```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 EXAMPLE.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

6. 客户端：`vim /etc/krb5.conf`

```
# Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
 default_realm = EXAMPLE.COM
 #default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 EXAMPLE.COM = {
  kdc = hadoop102
  admin_server = hadoop102
 }
[domain_realm]
# .example.com = EXAMPLE.COM
# example.com = EXAMPLE.COM
```

7. 初始化KDC数据库 `kdb5_util create -s`

8. 修改管理员权限配置文件 `vim /var/kerberos/krb5kdc/kadm5.acl`

```
*/admin@EXAMPLE.COM     *
```

9. 启动Kerberos相关服务,服务端  `systemctl start/enable krb5kdc` && `systemctl start/enable kadmin`

10. 创建Kerberos管理员用户 `kadmin.local -q "addprinc admin/admin"`

### Kerberos使用概述

1. Kerberos数据库操作

|操作|命令|
| :-- | :-- |
|shell|shell通过`kadmin.local -q ""`传递命令|
|登录数据库(本地登录，无需认证)|`kadmin.local`|
|登录数据库(远程登陆，需进行主体认证)|`kadmin`|
|退出|`exit`|
|创建Kerberos主体(登录数据库后)|`addprinc test`|
|创建Kerberos主体(shell命令)|`kadmin.local -q "addprinc test"`|
|修改主体密码(登录数据库后)|`cpw test`|
|查看所有主体(登录数据库后)|`list_principals`|
|密码认证(shell)|`kinit test`|
|查看认证凭证(shell)|`klist`|
|生成主题的keytab文件到/root/test.keytab(shell)|`kadmin.local -q "xst -k  /root/test.keytab test@EXAMPLE.COM"`|
|生成主题的keytab文件到/root/test.keytab(shell,使用norandkey：不随机生成密码)|`kadmin.local -q "xst -norandkey -k  /root/test.keytab test@EXAMPLE.COM`|
|使用keytab进行认证|`kinit -kt /root/test.keytab test`|
|销毁凭证(shell)|`kdestroy`|
 
### 创建Hadoop系统用户

1. 为Hadoop开启Kerberos，需为不同服务准备不同的用户，启动服务时需要使用相应的用户。须在所有节点创建以下用户和用户组。嫌麻烦可以直接root，应该可以?

|User:Group|Daemons|
| :-- | :-- |
|hdfs:hadoop|NameNode, Secondary NameNode, JournalNode, DataNode|
|yarn:hadoop|ResourceManager, NodeManager|
|mapred:hadoop|MapReduce JobHistory Server|

2. 创建hadoop组(所有节点) `groupadd hadoop`

3. 创建各用户并设置密码(所有节点) 

```shell
useradd hdfs -g hadoop
echo hdfs | passwd --stdin  hdfs
useradd yarn -g hadoop
echo yarn | passwd --stdin yarn
useradd mapred -g hadoop
echo mapred | passwd --stdin mapred
```

### Hadoop Kerberos 配置

0. 创建主体 -> 修改hadoop配置文件 -> 配置HDFS使用HTTPS安全传输协议 -> 配置Yarn使用LinuxContainerExcutor 

1. 为Hadoop各服务创建Kerberos主体（Principal）

    1. 主体格式如下: ServiceName/HostName@REALM，例如 dn/hadoop102@EXAMPLE.COM

    2. 各服务所需主体如下

    |服务|所在主机|主体（Principal）|
    | :-- | :-- | :-- |
    |NameNode|hadoop102|nn/hadoop102|
    |DataNode|hadoop102|dn/hadoop102|
    |DataNode|hadoop103	|dn/hadoop103|
    |DataNode|hadoop104|dn/hadoop104|
    |Secondary NameNode	|hadoop104|sn/hadoop104|
    |ResourceManager|hadoop103|rm/hadoop103|
    |NodeManager|hadoop102|nm/hadoop102|
    |NodeManager|hadoop103|nm/hadoop103|
    |NodeManager|hadoop104|nm/hadoop104|
    |JobHistory Server|hadoop102|jhs/hadoop102|
    |Web UI|hadoop102|HTTP/hadoop102|
    |Web UI|hadoop103|HTTP/hadoop103|
    |Web UI|hadoop104|HTTP/hadoop104|

    3. 创建主体说明

        1. 主体通过密钥文件keytab文件认证，需要一个路径存放keytab (在所有主机)
        
        ```shell
        mkdir /etc/security/keytab/
        chown -R root:hadoop /etc/security/keytab/
        chmod 770 /etc/security/keytab/
        ```

        2. 执行创建主体的语句，需登录Kerberos 数据库客户端 `kinit admin/admin` or 本地登录？

        3. 登录数据库客户端 `kadmin`

        4. 创建主体的语句 `addprinc -randkey test/test`

        5. 密钥写入keytab文件语句 `xst -k /etc/security/keytab/test.keytab test/test`

    4. 创建主体

        1. 执行3.1

        2. hadoop102操作

        ```shell
        kadmin -padmin/admin -wadmin -q"addprinc -randkey nn/hadoop102"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/nn.service.keytab nn/hadoop102"
        DataNode（hadoop102）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey dn/hadoop102"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/dn.service.keytab dn/hadoop102"
        NodeManager（hadoop102）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey nm/hadoop102"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/nm.service.keytab nm/hadoop102"
        JobHistory Server（hadoop102）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey jhs/hadoop102"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/jhs.service.keytab jhs/hadoop102"
        Web UI（hadoop102）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey HTTP/hadoop102"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/spnego.service.keytab HTTP/hadoop102"
        chmod 660 /etc/security/keytab/*
        ```

        3. hadoop103操作

        ```shell
        kadmin -padmin/admin -wadmin -q"addprinc -randkey rm/hadoop103"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/rm.service.keytab rm/hadoop103"
        DataNode（hadoop103）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey dn/hadoop103"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/dn.service.keytab dn/hadoop103"
        NodeManager（hadoop103）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey nm/hadoop103"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/nm.service.keytab nm/hadoop103"
        Web UI（hadoop103）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey HTTP/hadoop103"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/spnego.service.keytab HTTP/hadoop103"
        chmod 660 /etc/security/keytab/*
        ```

        4. hadoop104操作

        ```shell
        kadmin -padmin/admin -wadmin -q"addprinc -randkey dn/hadoop104"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/dn.service.keytab dn/hadoop104"
        Secondary NameNode（hadoop104）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey sn/hadoop104"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/sn.service.keytab sn/hadoop104"
        NodeManager（hadoop104）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey nm/hadoop104"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/nm.service.keytab nm/hadoop104"
        Web UI（hadoop104）
        kadmin -padmin/admin -wadmin -q"addprinc -randkey HTTP/hadoop104"
        kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/spnego.service.keytab HTTP/hadoop104"
        chmod 660 /etc/security/keytab/*
        ```

2. 修改Hadoop配置文件

    1. 需要修改的内容如下，修改完毕需要**分发所改文件**。

    2. `core-site.xml` 配置开启Kerberos主体到系统用户映射，设置映射规则，开启hadoop认证与授权，设置RPC通讯为仅认证

    ```xml
    <!-- Kerberos主体到系统用户的映射机制 -->
    <property>
        <name>hadoop.security.auth_to_local.mechanism</name>
        <value>MIT</value>
    </property>

    <!-- Kerberos主体到系统用户的具体映射规则 -->
    <property>
        <name>hadoop.security.auth_to_local</name>
        <value>
            RULE:[2:$1/$2@$0]([ndj]n\/.*@EXAMPLE\.COM)s/.*/hdfs/
            RULE:[2:$1/$2@$0]([rn]m\/.*@EXAMPLE\.COM)s/.*/yarn/
            RULE:[2:$1/$2@$0](jhs\/.*@EXAMPLE\.COM)s/.*/mapred/
            DEFAULT
        </value>
    </property>

    <!-- 启用Hadoop集群Kerberos安全认证 -->
    <property>
        <name>hadoop.security.authentication</name>
        <value>kerberos</value>
    </property>

    <!-- 启用Hadoop集群授权管理 -->
    <property>
        <name>hadoop.security.authorization</name>
        <value>true</value>
    </property>

    <!-- Hadoop集群间RPC通讯设为仅认证模式 -->
    <property>
        <name>hadoop.rpc.protection</name>
        <value>authentication</value>
    </property>
    ```

    3. `hdfs-site.xml` 开启Kerberos认证，配置各个服务的Kerberos主体和密钥路径

    ```xml
    <!-- 访问DataNode数据块时需通过Kerberos认证 -->
    <property>
    <name>dfs.block.access.token.enable</name>
    <value>true</value>
    </property>

    <!-- NameNode服务的Kerberos主体,_HOST会自动解析为服务所在的主机名 -->
    <property>
    <name>dfs.namenode.kerberos.principal</name>
    <value>nn/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- NameNode服务的Kerberos密钥文件路径 -->
    <property>
    <name>dfs.namenode.keytab.file</name>
    <value>/etc/security/keytab/nn.service.keytab</value>
    </property>

    <!-- Secondary NameNode服务的Kerberos主体 -->
    <property>
    <name>dfs.secondary.namenode.keytab.file</name>
    <value>/etc/security/keytab/sn.service.keytab</value>
    </property>

    <!-- Secondary NameNode服务的Kerberos密钥文件路径 -->
    <property>
    <name>dfs.secondary.namenode.kerberos.principal</name>
    <value>sn/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- NameNode Web服务的Kerberos主体 -->
    <property>
    <name>dfs.namenode.kerberos.internal.spnego.principal</name>
    <value>HTTP/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- WebHDFS REST服务的Kerberos主体 -->
    <property>
    <name>dfs.web.authentication.kerberos.principal</name>
    <value>HTTP/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- Secondary NameNode Web UI服务的Kerberos主体 -->
    <property>
    <name>dfs.secondary.namenode.kerberos.internal.spnego.principal</name>
    <value>HTTP/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- Hadoop Web UI的Kerberos密钥文件路径 -->
    <property>
    <name>dfs.web.authentication.kerberos.keytab</name>
    <value>/etc/security/keytab/spnego.service.keytab</value>
    </property>

    <!-- DataNode服务的Kerberos主体 -->
    <property>
    <name>dfs.datanode.kerberos.principal</name>
    <value>dn/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- DataNode服务的Kerberos密钥文件路径 -->
    <property>
    <name>dfs.datanode.keytab.file</name>
    <value>/etc/security/keytab/dn.service.keytab</value>
    </property>

    <!-- 配置NameNode Web UI 使用HTTPS协议 -->
    <property>
    <name>dfs.http.policy</name>
    <value>HTTPS_ONLY</value>
    </property>

    <!-- 配置DataNode数据传输保护策略为仅认证模式 -->
    <property>
    <name>dfs.data.transfer.protection</name>
    <value>authentication</value>
    </property>
    ```

    4. `yarn-site.xml` 配置各个服务的Kerberos主体和密钥路径

    ```xml
    <!-- Resource Manager 服务的Kerberos主体 -->
    <property>
    <name>yarn.resourcemanager.principal</name>
    <value>rm/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- Resource Manager 服务的Kerberos密钥文件 -->
    <property>
    <name>yarn.resourcemanager.keytab</name>
    <value>/etc/security/keytab/rm.service.keytab</value>
    </property>

    <!-- Node Manager 服务的Kerberos主体 -->
    <property>
    <name>yarn.nodemanager.principal</name>
    <value>nm/_HOST@EXAMPLE.COM</value>
    </property>

    <!-- Node Manager 服务的Kerberos密钥文件 -->
    <property>
    <name>yarn.nodemanager.keytab</name>
    <value>/etc/security/keytab/nm.service.keytab</value>
    </property>
    ```

    5. `mapred-site.xml` 配置各个服务的Kerberos主体和密钥路径

    ```xml
    <!-- 历史服务器的Kerberos主体 -->
    <property>
    <name>mapreduce.jobhistory.keytab</name>
    <value>/etc/security/keytab/jhs.service.keytab</value>
    </property>

    <!-- 历史服务器的Kerberos密钥文件 -->
    <property>
    <name>mapreduce.jobhistory.principal</name>
    <value>jhs/_HOST@EXAMPLE.COM</value>
    </property>
    ```

    6. 分发以上修改的配置文件,精确到文件



3. 配置HDFS使用HTTPS安全传输协议

    1. 生成密钥对,**密钥库的密码至少6个字符**，可以是纯数字或者字母或者数字和字母的组合

        1. Keytool是java数据证书的管理工具

        2. -keystore 指定密钥库的名称及位置

        3. -genkey(或者-genkeypair)生成密钥对

        4. -alias 为生成的密钥对指定别名，如果没有默认是mykey

        5. -keyalg  指定密钥的算法 RSA/DSA 默认是DSA

        ```shell
        keytool -keystore /etc/security/keytab/keystore -alias jetty -genkey -keyalg RSA
        ```
        
    2. 修改keystore文件的所有者和访问权限,**确保hdfs用户（HDFS的启动用户）具有对所生成keystore文件的读权限**

    ```shell
    chown -R root:hadoop /etc/security/keytab/keystore
    chmod 660 /etc/security/keytab/keystore
    ```

    3. 分发 /etc/security/keytab/keystore

    4. 修改hadoop配置文件ssl-server.xml.example,在hadoop配置文件中

    ```xml
    <!-- SSL密钥库路径 -->
    <property>
    <name>ssl.server.keystore.location</name>
    <value>/etc/security/keytab/keystore</value>
    </property>

    <!-- SSL密钥库密码 -->
    <property>
    <name>ssl.server.keystore.password</name>
    <value>123456</value>
    </property>

    <!-- SSL可信任密钥库路径 -->
    <property>
    <name>ssl.server.truststore.location</name>
    <value>/etc/security/keytab/keystore</value>
    </property>

    <!-- SSL密钥库中密钥的密码 -->
    <property>
    <name>ssl.server.keystore.keypassword</name>
    <value>123456</value>
    </property>

    <!-- SSL可信任密钥库密码 -->
    <property>
    <name>ssl.server.truststore.password</name>
    <value>123456</value>
    </property>
    ```

    5. 分发ssl-server.xml文件

4. 配置Yarn使用LinuxContainerExecutor

    1. 修改所有节点的`container-executor`所有者和权限，要求其所有者为root，所有组为hadoop（启动NodeManger的yarn用户的所属组），权限为6050。其默认路径为$HADOOP_HOME/bin

    ```shell
    chown root:hadoop /opt/module/hadoop-3.1.3/bin/container-executor
    chmod 6050 /opt/module/hadoop-3.1.3/bin/container-executor
    ```

    2. 修改所有节点的`container-executor.cfg`文件的所有者和权限，要求该文件及其所有的上级目录的所有者均为root，所有组为hadoop（启动NodeManger的yarn用户的所属组），权限为400。其默认路径为$HADOOP_HOME/etc/hadoop

    ```shell
    chown root:hadoop /opt/module/hadoop-3.1.3/etc/hadoop/container-executor.cfg
    chown root:hadoop /opt/module/hadoop-3.1.3/etc/hadoop
    chown root:hadoop /opt/module/hadoop-3.1.3/etc
    chown root:hadoop /opt/module/hadoop-3.1.3
    chown root:hadoop /opt/module
    chmod 400 /opt/module/hadoop-3.1.3/etc/hadoop/container-executor.cfg
    ```

    3. `vim $HADOOP_HOME/etc/hadoop/container-executor.cfg`

    ```shell
    yarn.nodemanager.linux-container-executor.group=hadoop
    banned.users=hdfs,yarn,mapred
    min.user.id=1000
    allowed.system.users=
    feature.tc.enabled=false
    ```

    4. `vim $HADOOP_HOME/etc/hadoop/yarn-site.xml`

    ```xml
    <!-- 配置Node Manager使用LinuxContainerExecutor管理Container -->
    <property>
    <name>yarn.nodemanager.container-executor.class</name>
    <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
    </property>

    <!-- 配置Node Manager的启动用户的所属组 -->
    <property>
    <name>yarn.nodemanager.linux-container-executor.group</name>
    <value>hadoop</value>
    </property>

    <!-- LinuxContainerExecutor脚本路径 -->
    <property>
    <name>yarn.nodemanager.linux-container-executor.path</name>
    <value>/opt/module/hadoop-3.1.3/bin/container-executor</value>
    </property>
    ```

### 安全模式下启动Hadoop

0. 感觉如果没有 创建Hadoop系统用户 的话，直接用root的话，**不要修改**。

1. 修改特定本地路径权限,高可用需要考虑全，下面只展示非高可用

    |||||
    | :-- | :-- | :-- | :-- | 
    |local|$HADOOP_LOG_DIR|hdfs:hadoop|drwxrwxr-x|
    |local|dfs.namenode.name.dir|hdfs:hadoop|drwx------|
    |local|dfs.datanode.data.dir|hdfs:hadoop|drwx------|
    |local|dfs.namenode.checkpoint.dir|hdfs:hadoop|drwx------|
    |local|yarn.nodemanager.local-dirs|yarn:hadoop|drwxrwxr-x|
    |local|yarn.nodemanager.log-dirs|yarn:hadoop|drwxrwxr-x|

    1. hadoop102

    ```shell
    chown hdfs:hadoop /opt/module/hadoop-3.1.3/logs/
    chmod 775 /opt/module/hadoop-3.1.3/logs/
    chown -R hdfs:hadoop /opt/module/hadoop-3.1.3/data/dfs/name/
    chmod 700 /opt/module/hadoop-3.1.3/data/dfs/name/
    chown -R hdfs:hadoop /opt/module/hadoop-3.1.3/data/dfs/data/
    chmod 700 /opt/module/hadoop-3.1.3/data/dfs/data/
    chown -R yarn:hadoop /opt/module/hadoop-3.1.3/data/nm-local-dir/
    chmod -R 775 /opt/module/hadoop-3.1.3/data/nm-local-dir/
    chown yarn:hadoop /opt/module/hadoop-3.1.3/logs/userlogs/
    chmod 775 /opt/module/hadoop-3.1.3/logs/userlogs/
    ```

    2. hadoop103

    ```shell
    chown hdfs:hadoop /opt/module/hadoop-3.1.3/logs/
    chmod 775 /opt/module/hadoop-3.1.3/logs/
    chown -R hdfs:hadoop /opt/module/hadoop-3.1.3/data/dfs/data/
    chmod 700 /opt/module/hadoop-3.1.3/data/dfs/data/
    chown -R yarn:hadoop /opt/module/hadoop-3.1.3/data/nm-local-dir/
    chmod -R 775 /opt/module/hadoop-3.1.3/data/nm-local-dir/
    chown yarn:hadoop /opt/module/hadoop-3.1.3/logs/userlogs/
    chmod 775 /opt/module/hadoop-3.1.3/logs/userlogs/
    ```

    3. hadoop104

    ```shell
    chown hdfs:hadoop /opt/module/hadoop-3.1.3/logs/
    chmod 775 /opt/module/hadoop-3.1.3/logs/
    chown -R hdfs:hadoop /opt/module/hadoop-3.1.3/data/dfs/data/
    chmod 700 /opt/module/hadoop-3.1.3/data/dfs/data/
    chown -R hdfs:hadoop /opt/module/hadoop-3.1.3/data/dfs/namesecondary/
    chmod 700 /opt/module/hadoop-3.1.3/data/dfs/namesecondary/
    chown -R yarn:hadoop /opt/module/hadoop-3.1.3/data/nm-local-dir/
    chmod -R 775 /opt/module/hadoop-3.1.3/data/nm-local-dir/
    chown yarn:hadoop /opt/module/hadoop-3.1.3/logs/userlogs/
    chmod 775 /opt/module/hadoop-3.1.3/logs/userlogs/
    ```

2. 启动HDFS

    1. sudo -i -u user password 命令 : -i表示重新加载环境变量；-u表示以特定用户身份执行后续命令

    2. 在主节点（hadoop102）配置**hdfs用户**到所有节点的免密登录

    3. 修改主节点（hadoop102）节点的$HADOOP_HOME/sbin/start-dfs.sh脚本，在**顶部**增加以下环境变量。stop-dfs.sh 同样

    ```shell
    HDFS_DATANODE_USER=hdfs
    HDFS_NAMENODE_USER=hdfs
    HDFS_SECONDARYNAMENODE_USER=hdfs
    ```

3. 修改HDFS特定路径访问权限(不存在要创建)

    |||||
    | :-- | :-- | :-- | :-- |
    |hdfs|	/|	hdfs:hadoop|	drwxr-xr-x|
    |hdfs|	/tmp|	hdfs:hadoop|	drwxrwxrwxt|
    |hdfs|	/user|	hdfs:hadoop	|drwxrwxr-x|
    |hdfs|	yarn.nodemanager.remote-app-log-dir|	yarn:hadoop	|drwxrwxrwxt|
    |hdfs|	mapreduce.jobhistory.intermediate-done-dir	|mapred:hadoop|	drwxrwxrwxt|
    |hdfs|	mapreduce.jobhistory.done-dir|	mapred:hadoop|	drwxrwx---|


    1. 创建hdfs/hadoop主体，执行以下命令并按照提示输入密码 `kadmin.local -q "addprinc hdfs/hadoop"`

    2. 认证 `kinit hdfs/hadoop`

    3. 修改权限

    ```shell
    hadoop fs -chown hdfs:hadoop / /tmp /user
    hadoop fs -chmod 755 /
    hadoop fs -chmod 1777 /tmp
    hadoop fs -chmod 775 /user
    hadoop fs -chown yarn:hadoop /tmp/logs
    hadoop fs -chmod 1777 /tmp/logs
    hadoop fs -chown -R mapred:hadoop /tmp/hadoop-yarn/staging/history/done_intermediate
    hadoop fs -chmod -R 1777 /tmp/hadoop-yarn/staging/history/done_intermediate

    hadoop fs -chown mapred:hadoop /tmp/hadoop-yarn/staging/history/
    hadoop fs -chown mapred:hadoop /tmp/hadoop-yarn/staging/
    hadoop fs -chown mapred:hadoop /tmp/hadoop-yarn/

    hadoop fs -chmod 770 /tmp/hadoop-yarn/staging/history/
    hadoop fs -chmod 770 /tmp/hadoop-yarn/staging/
    hadoop fs -chmod 770 /tmp/hadoop-yarn/
    hadoop fs -chown -R mapred:hadoop /tmp/hadoop-yarn/staging/history/done
    hadoop fs -chmod -R 750 /tmp/hadoop-yarn/staging/history/done

    hadoop fs -chown mapred:hadoop /tmp/hadoop-yarn/staging/history/
    hadoop fs -chown mapred:hadoop /tmp/hadoop-yarn/staging/
    hadoop fs -chown mapred:hadoop /tmp/hadoop-yarn/

    hadoop fs -chmod 770 /tmp/hadoop-yarn/staging/history/
    hadoop fs -chmod 770 /tmp/hadoop-yarn/staging/
    hadoop fs -chmod 770 /tmp/hadoop-yarn/
    ```

    4. 启动Yarn

        1. 在Yarn主节点（hadoop103）配置yarn用户到所有节点的免密登录

        2. 修改主节点（hadoop103）的$HADOOP_HOME/sbin/start-yarn.sh and stop-yarn.sh

        ```shell
        YARN_RESOURCEMANAGER_USER=yarn
        YARN_NODEMANAGER_USER=yarn
        ```

    5. 启动HistoryServer `sudo -i -u mapred mapred --daemon start historyserver`

### 安全集群使用说明

1. 用户要求,以下使用说明均基于普通用户，安全集群对用户有以下要求：

    1. 集群中的每个节点都需要创建该用户

    2. 该用户需要属于hadoop用户组

    3. 需要创建该用户对应的Kerberos主体 `kadmin -p admin/admin -wadmin -q"addprinc -pw user password"`

2. 访问HDFS集群文件

    1. 认证当前用户 `kinit user` -> 执行命令 -> 注销认证 `kdestory`

    2. web页面 

        1. 安装Kerberos客户端 http://web.mit.edu/kerberos/dist/kfw/4.1/kfw-4.1-amd64.msi　

        2. 编辑C:\ProgramData\MIT\Kerberos5\krb5.ini文件

        ```properties
        [libdefaults]
        dns_lookup_realm = false
        ticket_lifetime = 24h
        forwardable = true
        rdns = false
        default_realm = EXAMPLE.COM

        [realms]
        EXAMPLE.COM = {
        kdc = hadoop102
        admin_server = hadoop102
        }

        [domain_realm]
        ```

        3. 打开浏览器 输入 `about:config`

        4. 搜索`network.negotiate-auth.trusted-uris` 修改值为要访问的主机名（hadoop102）

        5. 搜索`network.auth.use-sspi`，双击将值变为false

        6. 启动Kerberos客户端，点击Get Ticket

        7. 输入主体名和密码，点击OK,认证成功后访问HDFS

3. 提交MapReduce任务：认证后，提交任务

### Hive用户认证配置

1. Hadoop集群启动Kerberos认证

2. 创建Hive系统用户和Kerberos主体（所有节点）

```shell
useradd hive -g hadoop
echo hive | passwd --stdin hive
```

3. 创建Kerberos主体并生成keytab文件 

```
kadmin -padmin/admin -wadmin -q"addprinc -randkey hive/hadoop102"
kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/hive.service.keytab hive/hadoop102"
chown -R root:hadoop /etc/security/keytab/
chmod 660 /etc/security/keytab/hive.service.keytab
```

4. 配置认证 $HIVE_HOME/conf/hive-site.xml

```xml
<!-- HiveServer2启用Kerberos认证 -->
<property>
    <name>hive.server2.authentication</name>
    <value>kerberos</value>
</property>

<!-- HiveServer2服务的Kerberos主体 -->
<property>
    <name>hive.server2.authentication.kerberos.principal</name>
    <value>hive/hadoop102@EXAMPLE.COM</value>
</property>

<!-- HiveServer2服务的Kerberos密钥文件 -->
<property>
    <name>hive.server2.authentication.kerberos.keytab</name>
    <value>/etc/security/keytab/hive.service.keytab</value>
</property>

<!-- Metastore启动认证 -->
<property>
    <name>hive.metastore.sasl.enabled</name>
    <value>true</value>
</property>
<!-- Metastore Kerberos密钥文件 -->
<property>
    <name>hive.metastore.kerberos.keytab.file</name>
    <value>/etc/security/keytab/hive.service.keytab</value>
</property>
<!-- Metastore Kerberos主体 -->
<property>
    <name>hive.metastore.kerberos.principal</name>
    <value>hive/hadoop102@EXAMPLE.COM</value>
</property>
```

5. $HADOOP_HOME/etc/hadoop/core-site.xml

```xml
<property>
    <name>hadoop.proxyuser.hive.hosts</name>
    <value>*</value>
</property>

<property>
    <name>hadoop.proxyuser.hive.groups</name>
    <value>*</value>
</property>

<property>
    <name>hadoop.proxyuser.hive.users</name>
    <value>*</value>
</property>
```

6. 分发

7. 重启hadoop

8. 启动hiveserver2 `sudo -i -u hive hiveserver2 > /dev/null 2>&1`

### Hive Kerberos 认证使用说明

1. 以下说明均基于普通用户

2. beeline 客户端: 认证，beeline连接，`jdbc:hive2://hadoop102:10000/;principal=hive/hadoop102@EXAMPLE.COM` 命令

3. DataGrip客户端

    1. 创建Driver，配置driver，名称，class（全类名），驱动包，url（jdbc:hive2://{host}:{port}/{database}[;<;,{:identifier}={:param}>]）

    2. 新建连接，配置连接
        
        1. 基础配置：host：hadoop102，url：`jdbc:hive2://hadoop102:10000/;principal=hive/hadoop102@EXAMPLE.COM`

        2. 高级配置：点击Advanced，在VM options中配置，user为自己认证的用户

        ```shell
        -Djava.security.krb5.conf="C:\\ProgramData\\MIT\\Kerberos5\\krb5.ini"
        -Djava.security.auth.login.config="C:\\ProgramData\\MIT\\Kerberos5\\user.conf"
        -Djavax.security.auth.useSubjectCredsOnly=false
        ```

        3. 编写JAAS（Java认证授权服务）配置文件,文件名和 `java.security.auth.login.config` 一致

        ```
        com.sun.security.jgss.initiate{
            com.sun.security.auth.module.Krb5LoginModule required
            useKeyTab=true
            useTicketCache=false
            keyTab="C:\\ProgramData\\MIT\\Kerberos5\\user.keytab"
            principal="user@EXAMPLE.COM";
        };
        ```

        4. 用户生成keytab文件，在krb5kdc所在节点（hadoop102）执行以下命令 `kadmin.local -q"xst -norandkey -k /home/user/user.keytab user"`

        5. 将上一步生成的atguigu.keytab文件，置于Windows中的特定路径，该路径须与3.中的keyTab属性的值**保持一致**

        6. 测试


## 安全环境实战(复杂得要死)

0. 基本流程：
    1. 创建相关用户(每个节点,用来启动服务) 
    2. 修改数据资源所有者为该用户
    3. 必要时改变hdfs文件权限(用认证的主体) 
    4. 创建主体(按需设置密码，如果自己用，或者接下来要用，设置，反之，可以不设置，而是设置keytab，比如nn/hadoop102没有设置密码，用来组件之间通信，所以设置了keytab在xml配置文件中，它们通过找到的keytab认证。比如sudo -i -u 时认证，通过keytab认证)  
    5. 配置用户到所有节点的ssh免密
    6. 启动时加入 -i -u 指定用户(必要时先指定主体认证，可以是别的主体)

1. Hadoop启用Kerberos安全认证之后，之前的非安全环境下的全流程调度脚本和即席查询引擎均会遇到认证问题，故需要对其进行改进

2. 此处统一将数仓的全部数据资源的所有者设为hive用户

    1. 为hive用户创建Keberos主体

    ```shell
    kadmin -padmin/admin -wadmin -q"addprinc -randkey hive"
    kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/hive.keytab hive"
    chown hive:hadoop /etc/security/keytab/hive.keytab
    chmod 440 /etc/security/keytab/hive.keytab
    ```

    2. 分发

3. 数据采集通道修改

    1. 用户行为日志 `kafka-flume-hdfs.conf`

    ```shell
    a1.sinks.k1.hdfs.kerberosPrincipal=hive@EXAMPLE.COM
    a1.sinks.k1.hdfs.kerberosKeytab=/etc/security/keytab/hive.keytab
    ```

    2. 业务数据 `mysql_to_hdfs.sh` 顶部增加如下认证语句

    ```shell
    kinit -kt /etc/security/keytab/hive.keytab hive
    ```

4. 数仓各层脚本修改,数仓各层脚本均需在顶部加入如下认证语句 `kinit -kt /etc/security/keytab/hive.keytab hive`,修改语句参考 `sed -i '1 a kinit -kt /etc/security/keytab/hive.keytab hive' hdfs_to_ods_log.sh`

5. 修改HDFS特定路径所有者

```shell
kinit hdfs/hadoop
hadoop fs -chown -R hive:hadoop /origin_data
hadoop fs -chown -R hive:hadoop /warehouse
hadoop fs -chown -R hive:hadoop /user/hive
hadoop fs -chown -R hive:hadoop /spark-history
```

6. Azkaban

    1. 各节点

    ```shell
    useradd azkaban -g hadoop
    echo azkaban | passwd --stdin azkaban
    ```

    2. 将各节点Azkaban安装路径所有者改为azkaban用户 `chown -R azkaban:hadoop /opt/module/azkaban`。使用azkaban用户启动Azkaban

    3. 启动Executor Server `udo -i -u azkaban bash -c "cd /opt/module/azkaban/azkaban-exec;bin/start-exec.sh"`

    4. 激活Executor Server `curl http://hadoop102:12321/executor?action=activate`

    5. 启动Web Server `sudo -i -u azkaban bash -c "cd /opt/module/azkaban/azkaban-web;bin/start-web.sh"`

    6. 修改数仓各层脚本访问权限，确保azkaban用户能够访问到 (各节点)

    ```shell
    chown -R atguigu:hadoop ${脚本所在目录}
    chmod 770 ${脚本所在目录}
    ```

7. 即席查询之Presto

    1. Presto集群开启Kerberos认证可只配置Presto Coordinator和Presto Cli之间进行认证。Hive Connector进行Kerberos认证

    2. 创建 presto 系统用户并创建presto主体，生成keytab文件，修改keytab访问权限，并分发

    ```shell
    kadmin -padmin/admin -wadmin -q"addprinc -randkey presto"
    kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/presto.keytab presto"
    chown presto:hadoop /etc/secukrity/keytab/presto.keytab
    xsync /etc/security/keytab/presto.keytab
    ```

    3. 为Presto Coordinator创建Kerberos主体:

    ```shell
    kadmin -padmin/admin -wadmin -q"addprinc -randkey presto/hadoop102"
    kadmin -padmin/admin -wadmin -q"xst -k /etc/security/keytab/presto.service.keytab presto/hadoop102"
    chown presto:hadoop /etc/security/keytab/presto.service.keytab
    ```

    4. 创建HTTPS协议所需的密钥对

        1. **alias**（别名）需要和Presto Coordinator的**Kerberos主体名保持一致**

        2. **名字与姓氏** 需要填写**Coordinator所在的主机名**

        3. `keytool -genkeypair -alias presto -keyalg RSA -keystore /etc/security/keytab/keystore.jks`

    5. 修改Presto Coordinator配置文件 `/opt/module/presto/etc/config.properties` 增加

    ```properties
    http-server.authentication.type=KERBEROS
    http.server.authentication.krb5.service-name=presto
    http.server.authentication.krb5.keytab=/etc/security/keytab/presto.service.keytab
    http.authentication.krb5.config=/etc/krb5.conf

    http-server.https.enabled=true
    http-server.https.port=7778
    http-server.https.keystore.path=/etc/security/keytab/keystore.jks
    http-server.https.keystore.key=123456
    ```

    6. 修改Hive Connector配置文件 `/opt/module/presto/etc/catalog/hive.properties` 增加

    ```properties
    hive.metastore.authentication.type=KERBEROS
    hive.metastore.service.principal=hive/hadoop102@EXAMPLE.COM
    hive.metastore.client.principal=presto@EXAMPLE.COM
    hive.metastore.client.keytab=/etc/security/keytab/presto.keytab

    hive.hdfs.authentication.type=KERBEROS
    hive.hdfs.impersonation.enabled=true
    hive.hdfs.presto.principal=presto@EXAMPLE.COM
    hive.hdfs.presto.keytab=/etc/security/keytab/presto.keytab
    hive.config.resources=/opt/module/hadoop-3.1.3/etc/hadoop/core-site.xml,/opt/module/hadoop-3.1.3/etc/hadoop/hdfs-site.xml
    ```

    7. 分发/opt/module/presto/etc/catalog/hive.properties

    8. 配置客户端Kerberos主体到用户名之间的映射规则

        1. 新建/opt/module/presto/etc/access-control.properties配置文件

        ```properties
        access-control.name=file
        security.config-file=etc/rules.json
        ```

        2. 新建/opt/module/presto/etc/rules.json文件
        
        ```properties
        {
        "catalogs": [
            {
            "allow": true
            }
        ],
        "user_patterns": [
            "(.*)",
            "([a-zA-Z]+)/?.*@.*"
        ]
        }
        ```
    
    9. 配置Presto代理用户

        1. 修改Hadoop配置文件 core-site.xml 增加,并分发，后重启hadoop

        ```xml
        <property>
            <name>hadoop.proxyuser.presto.hosts</name>
            <value>*</value>
        </property>

        <property>
            <name>hadoop.proxyuser.presto.groups</name>
            <value>*</value>
        </property>

        <property>
            <name>hadoop.proxyuser.presto.users</name>
            <value>*</value>
        </property>
        ```

        2. 重启presto

            1. 修改Presto安装路径所有者为presto

            ```shell
            chown -R presto:hadoop /opt/module/presto
            chown -R presto:hadoop /opt/module/presto
            chown -R presto:hadoop /opt/module/presto
            ```

            2. 使用hive用户启动MetaStore服务 `sudo -i -u hive hive --service metastore`

            3. 使用presto用户启动Presto服务 `sudo -i -u presto /opt/module/presto/bin/launcher start`
            
    10. 客户端认证访问Presto集群,使用普通用户访问，需要创建user(所有节点)，创建user主体，配置密码，认证，以下user由自己配置，当作以后访问服务的常用账户

    ```shell
    ./prestocli \
    --server https://hadoop102:7778 \
    --catalog hive \
    --schema default \
    --enable-authentication \
    --krb5-remote-service-name presto \
    --krb5-config-path /etc/krb5.conf \
    --krb5-principal user@EXAMPLE.COM \
    --krb5-keytab-path /home/user/user.keytab \
    --keystore-path /etc/security/keytab/keystore.jks \
    --keystore-password 123456 \
    --user user
    ```

8. 即席查询之Kylin

    1. 只需要具备一个Kerberos主体，进行常规的认证即可。但是Kylin所依赖的HBase需要进行额外的配置，才能在安全的Hadoop环境下正常工作。

    2. HBase开启Kerberos认证：创建用户hbase；（所有节点）创建主体hbase/hadoop102；生成密钥，在之前的目录；

    3. 修改HBase配置文件 `$HBASE_HOME/conf/hbase-site.xml`,分发

    ```xml
    <property>
    <name>hbase.security.authentication</name>
    <value>kerberos</value>
    </property>

    <property> 
    <name>hbase.master.kerberos.principal</name> 
    <value>hbase/_HOST@EXAMPLE.COM</value> 
    </property> 

    <property> 
    <name>hbase.master.keytab.file</name> 
    <value>/etc/security/keytab/hbase.service.keytab</value> 
    </property>

    <property>
    <name>hbase.regionserver.kerberos.principal</name> 
    <value>hbase/_HOST@EXAMPLE.COM</value> 
    </property> 

    <property> 
    <name>hbase.regionserver.keytab.file</name> 
    <value>/etc/security/keytab/hbase.service.keytab</value> 
    </property>

    <property> 
    <name>hbase.coprocessor.region.classes</name>
    <value>org.apache.hadoop.hbase.security.token.TokenProvider</value>
    </property>
    ```

    4. 修改hbase.rootdir路径所有者 

    ```shell
    kinit hdfs/hadoop
    hadoop fs -chown -R hbase:hadoop /hbase
    ```

    5. 修改hbase安装目录所有者(所有节点) `chown -R hbase:hadoop /opt/module/hbase`。配置hbase用户从主节点（hadoop102）到所有节点的ssh免密

    6. 使用hbase用户启动HBase `sudo -i -u hbase start-hbase.sh`

    7. 停止HBase `sudo -i -u hbase kinit -kt /etc/security/keytab/hbase.service.keytab hbase/hadoop102` && `sudo -i -u hbase stop-hbase.sh`

    8. Kylin进行Kerberos认证: 创建kylin系统用户;修改kylin.env.hdfs-working-dir路径所有者为kylin;用户认证 kinit hdfs/hadoop;修改所有者 hadoop fs -chown -R hive:hadoop /kylin; 修改/opt/module/kylin所有者为kylin `chown -R kylin:hadoop /opt/module/kylin`; 在kylin用户下认证为hive主体 `sudo -i -u kylin kinit -kt /etc/security/keytab/hive.keytab hive`; 以kylin用户的身份启动kylin `sudo -i -u kylin /opt/module/kylin/bin/kylin.sh start`

## 权限管理(可看)

### Ranger 概述

0. **Kerberos相当于系统用户管理，只有认证的用户可以登录。Ranger相当于对文件权限的管理，指定用户拥有rwx权限。**

1. Apache Ranger是一个Hadoop平台上的全方位**数据安全管理**框架，它可以为**整个Hadoop生态系统**提供全面的安全管理。

2. Ranger的目标

    1. 通过Ui或API进行集中化管理
    
    2. 通过一个工具对Hadoop体系进行细粒度授权

    3. 支持各个组件 授权认证标准

    4. 增强了不同业务场景的授权方法，基于角色或基于属性的授权

    5. 支持所有涉及安全的审计行为集中化管理

3. Ranger支持的框架

    * Apache Hadoop

    * Apache Hive
    * Apache HBase
    * Apache Storm
    * Apache Knox
    * Apache Solr
    * Apache Kafka
    * YARN
    * NIFI

4. Ranger的架构

    
    <span style="text-align: center;display:block;"><img src="./img/Ranger架构.png"/></span>

    1. Ranager的核心是Web应用程序，也称为**RangerAdmin**模块，此模块由**管理策略，审计日志和报告**等三部分组成

    2. 管理员通过 Web 或 API 设定策略。这些策略针对不同插件执行。

    3. solr 为 生成审计功能索引

    4. db存储 Linux机器上的用户和组信息

### Ranger安装

0. Ranger2.0要求对应的Hadoop为3.x以上，Hive为3.x以上版本，JDK为1.8以上版本。Hadoop及Hive等需开启用户认证功能，本文基于开启Kerberos安全认证的Hadoop和Hive环境。注：本文中所涉及的Ranger相关组件均安装在hadoop102节点。

1. 创建系统用户和Kerberos主体

    1. 创建ranger系统用户

    2. 检查HTTP主体 `kinit -kt /etc/security/keytab/spnego.service.keytab HTTP/hadoop102@EXAMPLE.COM`

    3. 创建rangeradmin主体(rangeradmin/hadoop102)/生成keytab文件/修改keytab文件所用这为ranger

    4. 创建rangerusersync主体(rangerusersync/hadoop102)和rangerlookup主体(rangerlookup/hadoop102)

2. 安装RangerAdmin

    1. 数据库环境准备

    ```sql
    create database ranger;
    grant all privileges on ranger.* to ranger@'%'  identified by 'ranger';
    ```

    2. 安装 RangerAdmin：上传tar.gz文件(ranger-2.0.0-admin.tar.gz)，解压到指定目录，对路径下的 install.properties 配置。

    ```properties
    #mysql驱动，根据自己路径配置，下同
    SQL_CONNECTOR_JAR=/opt/software/mysql-connector-java-5.1.48.jar

    #mysql的主机名和root用户的用户名密码，根据自己配置填写
    db_root_user=root
    db_root_password=000000
    db_host=hadoop102

    #ranger需要的数据库名和用户信息，和2.2.1创建的信息要一一对应
    db_name=ranger
    db_user=ranger
    db_password=ranger

    #Ranger各组件的admin用户密码，自己设置
    rangerAdmin_password=ciaran123
    rangerTagsync_password=ciaran123
    rangerUsersync_password=ciaran123
    keyadmin_password=ciaran123

    #ranger存储审计日志的路径，默认为solr，这里为了方便暂不设置
    audit_store=

    #策略管理器的url,rangeradmin安装在哪台机器，主机名就为对应的主机名
    policymgr_external_url=http://hadoop102:6080

    #启动ranger admin进程的linux用户信息
    unix_user=ranger
    unix_user_pwd=ranger
    unix_group=ranger

    #Kerberos相关配置
    spnego_principal=HTTP/hadoop102@EXAMPLE.COM
    spnego_keytab=/etc/security/keytab/spnego.service.keytab
    admin_principal=rangeradmin/hadoop102@EXAMPLE.COM
    admin_keytab=/etc/security/keytab/rangeradmin.keytab
    lookup_principal=rangerlookup/hadoop102@EXAMPLE.COM
    lookup_keytab=/etc/security/keytab/rangerlookup.keytab
    hadoop_conf=/opt/module/hadoop-3.1.3/etc/hadoop
    ```

    3. 在${RANGER_HOME} 下 执行安装脚本 `./setup.sh`

    4. 修改${RANGER_HOME}/conf/ranger-admin-site.xml

    ```xml
    <property>
        <name>ranger.jpa.jdbc.password</name>
        <value>ranger</value>
        <description />
    </property>

    <property>
        <name>ranger.service.host</name>
        <value>hadoop102</value>
    </property>
    ```

3. 启动RangerAdmin(以ranger用户启动) `sudo -i -u ranger ranger-admin start`

4. 访问Ranger的WebUI，地址为：http://hadoop102:6080

5. 停止ranger（此处不用执行）`sudo -i -u ranger ranger-admin stop`

6. 登录Ranger(默认可以使用用户名：admin，密码为之前配置的ciaran123进行登录！)

### RangerUsersync安装

0. RangerUsersync作为Ranger提供的一个管理模块，可以将Linux机器上的用户和组信息同步到RangerAdmin的数据库中进行管理

1. 解压tar.gz 文件(ranger-2.0.0-usersync.tar.gz)到指定目录，修改 `install.properties`

```properties
#rangeradmin的url
POLICY_MGR_URL =http://hadoop102:6080

#同步间隔时间，单位(分钟)
SYNC_INTERVAL = 1

#运行此进程的linux用户
unix_user=ranger
unix_group=ranger

#rangerUserSync用户的密码，参考rangeradmin中install.properties的配置
rangerUsersync_password=ciaran123

#Kerberos相关配置
usersync_principal=rangerusersync/hadoop102@EXAMPLE.COM
usersync_keytab=/etc/security/keytab/rangerusersync.keytab
hadoop_conf=/opt/module/hadoop-3.1.3/etc/hadoop
```

2. 在${RangerUsersync_HOME} 下 执行安装脚本 `./setup.sh`

3. 在${RangerUsersync_HOME}/conf/ 下 修改 `ranger-ugsync-site.xml`

```xml
<property>
      <name>ranger.usersync.enabled</name>
      <value>true</value>
</property>
```

4. 启动RangerUserSync `sudo -i -u ranger ranger-usersync start`

### Ranger Hive-plugin安装

0. Ranger Hive-plugin是Ranger对hive进行权限管理的插件。需要注意的是，Ranger Hive-plugin只能对使用jdbc方式访问hive的请求进行权限管理，hive-cli并不受限制。

1. 解压tar.gz文件(ranger-2.0.0-hive-plugin.tar.gz)，配置 `install.properties`

```properties
#策略管理器的url地址
POLICY_MGR_URL=http://hadoop102:6080

#组件名称
REPOSITORY_NAME=hive

#hive的安装目录
COMPONENT_INSTALL_DIR_NAME=/opt/module/hive

#hive组件的启动用户
CUSTOM_USER=hive

#hive组件启动用户所属组
CUSTOM_GROUP=hadoop
```

2. 用Ranger Hive-plugin `./enable-hive-plugin.sh`

3. 查看$HIVE_HOME/conf目录是否出现以下配置文件，如出现则表示Hive插件启用成功。 `ls $HIVE_HOME/conf | grep -E hiveserver2\|ranger`

```xml
hiveserver2-site.xml
ranger-hive-audit.xml
ranger-hive-security.xml
ranger-policymgr-ssl.xml
ranger-security.xml
```

4. 重启Hiveserver2，需使用hive用户启动 `sudo -i -u hive hiveserver2`

5. 在ranger admin上配置hive插件: 点击settings/user/点击hive用户/设置select Role为admin

6. 配置Hive插件：access Manager/hive框右边的+号/配置service name（与hive-plugin的install.properties一致）；username，password，hvie jdbc驱动类，hive jdbc url。参考如下。之后保存

```
Service Name：hive
Username：rangerlookup
Password：rangerlookup
jdbc.driverClassName：org.apache.hive.jdbc.HiveDriver
jdbc.url：jdbc:hive2://hadoop102:10000/;principal=hive/hadoop102@EXAMPLE.COM
```

7. 保存后，点击hive框下的hive服务/重新编辑/test connection/连接成功，则正常

### 对Hive进行权限管理

0. 点击hive框下的hive服务中的用户有访问hive资源的权限，其他用户没有

1. 赋予用户对gmall数据库的访问权限：进入hive服务/点击Add New Policy/设置授权策略/策略名称，database，table，column/允许条件中设置授权用户（select user中选择用户，permissions设置权限）/拒绝条件中设置拒绝用户。最后添加

2. 该用户在Kerberos认证后有权限访问hive中的有权限的数据库，数据表和数据列。

3. **Ranger授权模型**

    1. Ranger所采用的权限管理模型可归类为RBAC（Role-Based Access Control ）基于角色的访问控制

    2. 基础的RBAC模型共包含三个实体，分别是用户（user）、角色（role）和权限（permission）

    3. **用户需划分为某个角色**，权限的授予对象也是角色，例如用户张三为管理角色，那他就拥有了管理员角色的所有权限。


### 其他权限管理

0. https://cwiki.apache.org/confluence/display/RANGER/Row-level+filtering+and+column-masking+using+Apache+Ranger+policies+in+Apache+Hive

## 元数据管理(可看)

1. Apache Atlas为组织提供开放式元数据管理和治理功能。Atlas的具体功能如下：

|||
| :-- | :-- |
|元数据分类|支持对元数据进行分类管理，例如个人信息，敏感信息等|
|元数据检索|可按照元数据类型、元数据分类进行检索，支持全文检索|
|血缘依赖|支持表到表和字段到字段之间的血缘依赖，便于进行问题回溯和影响分析等|

2. 表与表之间的血缘依赖

<span style="text-align: center;display:block;"><img src="./img/Atlas-表与表之间的血缘关系.png"/></span>

3. 字段与字段之间的血缘依赖

<span style="text-align: center;display:block;"><img src="./img/Atlas-字段与字段之间的血缘依赖.png"/></span>

4. Atlas 架构原理

<span style="text-align: center;display:block;"><img src="./img/Atlas架构原理.png"/></span>

### Atlas安装

1. Atlas官网地址：https://atlas.apache.org/

    1. 文档查看地址：https://atlas.apache.org/2.1.0/index.html

    2. 下载地址：https://www.apache.org/dyn/closer.cgi/atlas/2.1.0/apache-atlas-2.1.0-sources.tar.gz 

2. 集成外部的HBase + Solr

3. solr安装 见 [Solr简介]

4. 安装Atlas

    1. 解压 apache-atlas-2.1.0-server.tar.gz ，更名

    2. 修改/opt/module/atlas/conf/atlas-application.properties配置文件中的以下参数

    ```
    atlas.graph.storage.hostname=hadoop102:2181,hadoop103:2181,hadoop104:2181
    ```

    3. 修改/opt/module/atlas/conf/atlas-env.sh配置文件，增加以下内容 `export HBASE_CONF_DIR=/opt/module/hbase/conf`
    
    4. 修改/opt/module/atlas/conf/atlas-application.properties配置文件中的以下参数

    ```
    atlas.graph.index.search.backend=solr
    atlas.graph.index.search.solr.mode=cloud
    atlas.graph.index.search.solr.zookeeper-url=hadoop102:2181,hadoop103:2181,hadoop104:2181
    ```

    5. 创建solr collection

    ```shell
    sudo -i -u solr /opt/module/solr/bin/solr create  -c vertex_index -d /opt/module/atlas/conf/solr -shards 3 -replicationFactor 2
    sudo -i -u solr /opt/module/solr/bin/solr create -c edge_index -d /opt/module/atlas/conf/solr -shards 3 -replicationFactor 2
    sudo -i -u solr /opt/module/solr/bin/solr create -c fulltext_index -d /opt/module/atlas/conf/solr -shards 3 -replicationFactor 2
    ```

5. Atlas集成Kafka

    1. 修改/opt/module/atlas/conf/atlas-application.properties配置文件中的以下参数

    ```
    atlas.notification.embedded=false
    atlas.kafka.data=/opt/module/kafka/data
    atlas.kafka.zookeeper.connect= hadoop102:2181,hadoop103:2181,hadoop104:2181/kafka
    atlas.kafka.bootstrap.servers=hadoop102:9092,hadoop103:9092,hadoop104:9092
    ```

6. Atlas Server配置

    1. 修改/opt/module/atlas/conf/atlas-application.properties配置文件中的以下参数

    ```
    #########  Server Properties  #########
    atlas.rest.address=http://hadoop102:21000
    # If enabled and set to true, this will run setup steps when the server starts
    atlas.server.run.setup.on.start=false

    #########  Entity Audit Configs  #########
    atlas.audit.hbase.zookeeper.quorum=hadoop102:2181,hadoop103:2181,hadoop104:2181
    ```

    2. 记录性能指标，进入/opt/module/atlas/conf/路径，修改当前目录下的atlas-log4j.xml

    ```xml
    #去掉如下代码的注释
    <appender name="perf_appender" class="org.apache.log4j.DailyRollingFileAppender">
        <param name="file" value="${atlas.log.dir}/atlas_perf.log" />
        <param name="datePattern" value="'.'yyyy-MM-dd" />
        <param name="append" value="true" />
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%d|%t|%m%n" />
        </layout>
    </appender>

    <logger name="org.apache.atlas.perf" additivity="false">
        <level value="debug" />
        <appender-ref ref="perf_appender" />
    </logger>
    ```

7. Kerberos相关配置

    0. 若Hadoop集群开启了Kerberos认证，Atlas与Hadoop集群交互之前就需要先进行Kerberos认证

    1. 为Atlas创建Kerberos主体，并生成keytab文件(当然也可以创建atlas用户用来专门启动atlas)

    2. 修改/opt/module/atlas/conf/atlas-application.properties配置文件，增加以下参数

    ```
    atlas.authentication.method=kerberos
    atlas.authentication.principal=atlas/hadoop102@EXAMPLE.COM
    atlas.authentication.keytab=/etc/security/keytab/atlas.service.keytab
    ```

8. Atlas集成Hive

    1. 安装Hive Hook: 解压 apache-atlas-2.1.0-hive-hook.tar.gz到atlas安装路径

    2. 修改/opt/module/hive/conf/hive-env.sh配置文件，增加如下参数 `export HIVE_AUX_JARS_PATH=/opt/module/atlas/hook/hive`

    3. 修改Hive配置文件 `hive-site.xml`

    ```xml
    <property>
        <name>hive.exec.post.hooks</name>
        <value>org.apache.atlas.hive.hook.HiveHook</value>
    </property>
    ```

    4. 修改/opt/module/atlas/conf/atlas-application.properties配置文件中的以下参数

    ```
    ######### Hive Hook Configs #######
    atlas.hook.hive.synchronous=false
    atlas.hook.hive.numRetries=3
    atlas.hook.hive.queueSize=10000
    atlas.cluster.name=primary
    ```

    5. 将Atlas配置文件/opt/module/atlas/conf/atlas-application.properties,拷贝到/opt/module/hive/conf目录

9. Atlas启动

    0. 错误信息查看路径：/opt/module/atlas/logs/*.out和application.log。停止Atlas服务命令为atlas_stop.py

    1. 启动Hadoop集群，zookeeper，kafka，hbase，solr

    2. 进入atlas安装路径 `bin/atlas_start.py`

10. 访问Atlas的WebUI：访问地址：http://hadoop102:21000 （账户：admin密码：admin）

### Atlas使用

0. 主要工作是同步各服务（主要是Hive）的元数据，并构建元数据实体之间的关联关系，然后对所存储的元数据建立索引，最终为用户提供数据血缘查看及元数据检索等功能。

1. Atlas在安装之初，需手动执行一次元数据的全量导入，后续Atlas便会利用Hive Hook增量同步Hive的元数据

2. Hive元数据初次导入

    1. 导入Hive元数据 `/opt/module/atlas/hook-bin/import-hive.sh` 账号密码：admin

    2. 查看hive元数据，web页面搜索hive_table类型

    3. 任选一张表查看血缘依赖关系

3. Hive元数据增量同步，无需人工调度，会通过DDL，DML获取数据间的血源关系

### 扩展

1. Atlas源码编译

2. 安装Maven

    1. Maven下载：https://maven.apache.org/download.cgi

    2. 把apache-maven-3.6.1-bin.tar.gz上传到linux的/opt/software目录下

    3. 解压apache-maven-3.6.1-bin.tar.gz到/opt/module/目录下面。修改名称为maven。添加环境变量到/etc/profile中

    4. 测试安装结果 `mvn -v`

    5. 修改setting.xml，指定为阿里云 `vim /opt/module/maven/conf/settings.xml`

    ```xml
    <!-- 添加阿里云镜像-->
    <mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>central</mirrorOf>
        <name>Nexus aliyun</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
    <mirror>
        <id>UK</id>
        <name>UK Central</name>
        <url>http://uk.maven.org/maven2</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
        <id>repo1</id>
        <mirrorOf>central</mirrorOf>
        <name>Human Readable Name for this Mirror.</name>
        <url>http://repo1.maven.org/maven2/</url>
    </mirror>
    <mirror>
        <id>repo2</id>
        <mirrorOf>central</mirrorOf>
        <name>Human Readable Name for this Mirror.</name>
        <url>http://repo2.maven.org/maven2/</url>
    </mirror>
    ```

3. 编译Atlas源码。上传源码包，解压，执行maven语句

```shell
export MAVEN_OPTS="-Xms2g -Xmx2g"
cd /opt/module/apache-atlas-sources-2.1.0/
mvn clean -DskipTests install
mvn clean -DskipTests package -Pdis
# 一定要在${atlas_home}执行
cd distro/target/
mv apache-atlas-2.1.0-server.tar.gz /opt/software/
mv apache-atlas-2.1.0-hive-hook.tar.gz /opt/software/
```

4. Atlas内存配置。以下是常见的服务器端选项。修改配置文件/`opt/module/atlas/conf/atlas-env.sh`

```shell
#设置Atlas内存
export ATLAS_SERVER_OPTS="-server -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+PrintTenuringDistribution -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=dumps/atlas_server.hprof -Xloggc:logs/gc-worker.log -verbose:gc -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=1m -XX:+PrintGCDetails -XX:+PrintHeapAtGC -XX:+PrintGCTimeStamps"

#建议JDK1.7使用以下配置
export ATLAS_SERVER_HEAP="-Xms15360m -Xmx15360m -XX:MaxNewSize=3072m -XX:PermSize=100M -XX:MaxPermSize=512m"

#建议JDK1.8使用以下配置
export ATLAS_SERVER_HEAP="-Xms15360m -Xmx15360m -XX:MaxNewSize=5120m -XX:MetaspaceSize=100M -XX:MaxMetaspaceSize=512m"

#如果是Mac OS用户需要配置
export ATLAS_SERVER_OPTS="-Djava.awt.headless=true -Djava.security.krb5.realm= -Djava.security.krb5.kdc="
```

5. 配置用户名密码。Atlas支持以下身份验证方法：File、Kerberos协议、LDAP协议

    1. 通过修改配置文件atlas-application.properties文件开启或关闭三种验证方法

    ```properties
    atlas.authentication.method.kerberos=true|false
    atlas.authentication.method.ldap=true|false
    atlas.authentication.method.file=true|false
    ```

    2. 如果两个或多个身份证验证方法设置为true，如果较早的方法失败，则身份验证将回退到后一种方法
    
    3. 文件方式修改用户名和密码设置 vim /opt/module/atlas/conf/users-credentials.properties，执行 `echo -n "ciaran"|sha256sum`sha156加密的密码

    ```
    #username=group::sha256-password
    # admin 为 用户名称，后面的一串时admin通过sha156加密后的
    # 获取sha256加密的用户密码。如ciaran，将admin改为ciaran，将ADMIN::后面的一串改为获得的加密字符串
    admin=ADMIN::8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
    rangertagsync=RANGER_TAG_SYNC::e3f67240f5117d1753c940dae9eea772d36ed5fe9bd9c94a300e40413f1afb9d
    ```

## 数据质量

### 数据质量管理概述

0. 数据质量管理定义：监控各个阶段的各类数据问题，识别，度量，监控，预警等管理活动。

1. 数据质量评价指标

|评价标准|描述|监控项|
| :-- | :-- | :-- |
|唯一性|指主键保持唯一|字段唯一性检查|
|完整性|主要包括记录缺失和字段值缺失等方面|字段枚举值检查/字段记录数检查/字段空值检查|
|精确度|数据生成的正确性，数据在整个链路流转的正确性|波动阀值检查|
|合法性|主要包括格式、类型、域值的合法性|字段日期格式检查/字段长度检查/字段值域检查|
|时效性|主要包括数据处理的时效性|批处理是否按时完成|

### 数据质量管理实操

1. 需求：主要监控以下数据的指标：

    1. ODS层数据量，每日环比和每周同比变化不能超过一定范围

    2. DIM层不能出现id空值，重复值；

    3. DWD层不能出现id空值，重复值；

2. 功能模块

<span style="text-align: center;display:block;">
<img src="./img/数据质量管理-功能模块.png"/></span>

3. 开发环境准备

    1. IDEA 安装 Python 插件，新建Python项目

    2. 在Project Structure中add SDK，点击SSH Interpreter，选择 Existing server configuration，点击+，填入服务器host，user name，password。

    3. 在mysql中创建 data_supervisor 库

    ```sql
    drop database if exists data_supervisor;
    create database data_supervisor;
    -- 创建空值指标表，null_id
    CREATE TABLE data_supervisor.`null_id`
    (
        `dt`                 date        NOT NULL COMMENT '日期',
        `tbl`                varchar(50) NOT NULL COMMENT '表名',
        `col`                varchar(50) NOT NULL COMMENT '列名',
        `value`              int         DEFAULT NULL COMMENT '空ID个数',
        `value_min`          int         DEFAULT NULL COMMENT '下限',
        `value_max`          int         DEFAULT NULL COMMENT '上限',
        `notification_level` int         DEFAULT NULL COMMENT '警告级别',
        PRIMARY KEY (`dt`, `tbl`, `col`)
    ) ENGINE = InnoDB
    DEFAULT CHARSET = utf8
        comment '空值指标表';
    --创建重复值指标表，duplicate
    CREATE TABLE data_supervisor.`duplicate`
    (
        `dt`                 date        NOT NULL COMMENT '日期',
        `tbl`                varchar(50) NOT NULL COMMENT '表名',
        `col`                varchar(50) NOT NULL COMMENT '列名',
        `value`              int         DEFAULT NULL COMMENT '重复值个数',
        `value_min`          int         DEFAULT NULL COMMENT '下限',
        `value_max`          int         DEFAULT NULL COMMENT '上限',
        `notification_level` int         DEFAULT NULL COMMENT '警告级别',
        PRIMARY KEY (`dt`, `tbl`, `col`)
    ) ENGINE = InnoDB
    DEFAULT CHARSET = utf8
        comment '重复值指标表';
    --创建值域指标表，rng
    CREATE TABLE data_supervisor.`rng`
    (
        `dt`                 date        NOT NULL COMMENT '日期',
        `tbl`                varchar(50) NOT NULL COMMENT '表名',
        `col`                varchar(50) NOT NULL COMMENT '列名',
        `value`              int         DEFAULT NULL COMMENT '超出预定值域个数',
        `range_min`          int         DEFAULT NULL COMMENT '值域下限',
        `range_max`          int         DEFAULT NULL COMMENT '值域上限',
        `value_min`          int         DEFAULT NULL COMMENT '下限',
        `value_max`          int         DEFAULT NULL COMMENT '上限',
        `notification_level` int         DEFAULT NULL COMMENT '警告级别',
        PRIMARY KEY (`dt`, `tbl`, `col`)
    ) ENGINE = InnoDB
    DEFAULT CHARSET = utf8
        comment '值域指标表';
    --创建环比增长指标表，day_on_day
    CREATE TABLE data_supervisor.`day_on_day`
    (
        `dt`                 date        NOT NULL COMMENT '日期',
        `tbl`                varchar(50) NOT NULL COMMENT '表名',
        `value`              double DEFAULT NULL COMMENT '环比增长百分比',
        `value_min`          double DEFAULT NULL COMMENT '增长上限',
        `value_max`          double DEFAULT NULL COMMENT '增长上限',
        `notification_level` int    DEFAULT NULL COMMENT '警告级别',
        PRIMARY KEY (`dt`, `tbl`)
    ) ENGINE = InnoDB
    DEFAULT CHARSET = utf8
        comment '环比增长指标表';
    --创建同比增长指标表，week_on_week
    CREATE TABLE data_supervisor.`week_on_week`
    (
        `dt`                 date        NOT NULL COMMENT '日期',
        `tbl`                varchar(50) NOT NULL COMMENT '表名',
        `value`              double DEFAULT NULL COMMENT '同比增长百分比',
        `value_min`          double DEFAULT NULL COMMENT '增长上限',
        `value_max`          double DEFAULT NULL COMMENT '增长上限',
        `notification_level` int    DEFAULT NULL COMMENT '警告级别',
        PRIMARY KEY (`dt`, `tbl`)
    ) ENGINE = InnoDB
    DEFAULT CHARSET = utf8
        comment '同比增长指标表';
    ```

#### 规则检测模块

1. 单一规则检测脚本编写

    1. 空id检查脚本(计算空值个数，并将结果和自己定义的阈值上下限，插入到MySQL) LF格式。null_id.sh
    
    ```shell
    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-
    # 检查id空值
    # 解析参数
    while getopts "t:d:c:s:x:l:" arg; do
    case $arg in
    # 要处理的表名
    t)
        TABLE=$OPTARG
        ;;
    # 日期
    d)
        DT=$OPTARG
        ;;
    # 要计算空值的列名
    c)
        COL=$OPTARG
        ;;
    # 空值指标下限
    s)
        MIN=$OPTARG
        ;;
    # 空值指标上限
    x)
        MAX=$OPTARG
        ;;
    # 告警级别
    l)
        LEVEL=$OPTARG
        ;;
    ?)
        echo "unkonw argument"
        exit 1
        ;;
    esac
    done

    #如果dt和level没有设置，那么默认值dt是昨天 告警级别是0
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)
    [ "$LEVEL" ] || LEVEL=0

    # 数仓DB名称
    HIVE_DB=gmall

    # 查询引擎
    HIVE_ENGINE=hive

    # MySQL相关配置
    mysql_user="root"
    mysql_passwd="000000"
    mysql_host="hadoop102"
    mysql_DB="data_supervisor"
    mysql_tbl="null_id"

    # 认证为hive用户，如在非安全(Hadoop未启用Kerberos认证)环境中，则无需认证
    kinit -kt /etc/security/keytab/hive.keytab hive

    # 空值个数
    RESULT=$($HIVE_ENGINE -e "set hive.cli.print.header=false;select count(1) from $HIVE_DB.$TABLE where dt='$DT' and $COL is null;")

    #结果插入MySQL
    mysql -h"$mysql_host" -u"$mysql_user" -p"$mysql_passwd" \
    -e"INSERT INTO $mysql_DB.$mysql_tbl VALUES('$DT', '$TABLE', '$COL', $RESULT, $MIN, $MAX, $LEVEL)
    ON DUPLICATE KEY UPDATE \`value\`=$RESULT, value_min=$MIN, value_max=$MAX, notification_level=$LEVEL;"
    ```

    2. 重复id检查脚本(计算重复值个数，并将结果和自己定义的阈值上下限，插入到MySQL表中。)。duplicate.sh

    ```shell
    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-
    # 监控某张表一列的重复值
    # 参数解析
    while getopts "t:d:c:s:x:l:" arg; do
    case $arg in
    # 要处理的表名
    t)
        TABLE=$OPTARG
        ;;
    # 日期
    d)
        DT=$OPTARG
        ;;
    # 要计算重复值的列名
    c)
        COL=$OPTARG
        ;;
    # 重复值指标下限
    s)
        MIN=$OPTARG
        ;;
    # 重复值指标上限
    x)
        MAX=$OPTARG
        ;;
    # 告警级别
    l)
        LEVEL=$OPTARG
        ;;
    ?)
        echo "unkonw argument"
        exit 1
        ;;
    esac
    done

    #如果dt和level没有设置，那么默认值dt是昨天 告警级别是0
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)
    [ "$LEVEL" ] || LEVEL=0

    # 数仓DB名称
    HIVE_DB=gmall

    # 查询引擎
    HIVE_ENGINE=hive

    # MySQL相关配置
    mysql_user="root"
    mysql_passwd="000000"
    mysql_host="hadoop102"
    mysql_DB="data_supervisor"
    mysql_tbl="duplicate"

    # 认证为hive用户，如在非安全(Hadoop未启用Kerberos认证)环境中，则无需认证
    kinit -kt /etc/security/keytab/hive.keytab hive

    # 重复值个数
    RESULT=$($HIVE_ENGINE -e "set hive.cli.print.header=false;select count(1) from (select $COL from $HIVE_DB.$TABLE where dt='$DT' group by $COL having count($COL)>1) t1;")

    # 将结果插入MySQL
    mysql -h"$mysql_host" -u"$mysql_user" -p"$mysql_passwd" \
    -e"INSERT INTO $mysql_DB.$mysql_tbl VALUES('$DT', '$TABLE', '$COL', $RESULT, $MIN, $MAX, $LEVEL)
    ON DUPLICATE KEY UPDATE \`value\`=$RESULT, value_min=$MIN, value_max=$MAX, notification_level=$LEVEL;"
    ```

    3. 值域检查脚本(计算超出规定值域的值的个数，并将结果和自己定义的阈值上下限，插入到MySQL表中。) range.sh

    ```shell
    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-
    # 计算某一列异常值个数

    while getopts "t:d:l:c:s:x:a:b:" arg; do
    case $arg in
    # 要处理的表名
    t)
        TABLE=$OPTARG
        ;;
    # 日替
    d)
        DT=$OPTARG
        ;;
    # 要处理的列
    c)
        COL=$OPTARG
        ;;
    # 不在规定值域的值的个数下限
    s)
        MIN=$OPTARG
        ;;
    # 不在规定值域的值的个数上限
    x)
        MAX=$OPTARG
        ;;
    # 告警级别
    l)
        LEVEL=$OPTARG
        ;;
    # 规定值域为a-b
    a)
        RANGE_MIN=$OPTARG
        ;;
    b)
        RANGE_MAX=$OPTARG
        ;;
    ?)
        echo "unkonw argument"
        exit 1
        ;;
    esac
    done

    #如果dt和level没有设置，那么默认值dt是昨天 告警级别是0
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)
    [ "$LEVEL" ] || LEVEL=0

    # 数仓DB名称
    HIVE_DB=gmall

    # 查询引擎
    HIVE_ENGINE=hive

    # MySQL相关配置
    mysql_user="root"
    mysql_passwd="000000"
    mysql_host="hadoop102"
    mysql_DB="data_supervisor"
    mysql_tbl="rng"

    # 认证为hive用户，如在非安全(Hadoop未启用Kerberos认证)环境中，则无需认证
    kinit -kt /etc/security/keytab/hive.keytab hive

    # 查询不在规定值域的值的个数
    RESULT=$($HIVE_ENGINE -e "set hive.cli.print.header=false;select count(1) from $HIVE_DB.$TABLE where dt='$DT' and $COL not between $RANGE_MIN and $RANGE_MAX;")

    # 将结果写入MySQL
    mysql -h"$mysql_host" -u"$mysql_user" -p"$mysql_passwd" \
    -e"INSERT INTO $mysql_DB.$mysql_tbl VALUES('$DT', '$TABLE', '$COL', $RESULT, $RANGE_MIN, $RANGE_MAX, $MIN, $MAX, $LEVEL)
    ON DUPLICATE KEY UPDATE \`value\`=$RESULT, range_min=$RANGE_MIN, range_max=$RANGE_MAX, value_min=$MIN, value_max=$MAX, notification_level=$LEVEL;"
    ```

    4. 数据量环比检查脚本(计算数据量环比增长值，并将结果和自己定义的阈值上下限，插入到MySQL表中。) day_on_day.sh

    ```shell
    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-
    # 计算一张表单日数据量环比增长值
    # 参数解析
    while getopts "t:d:s:x:l:" arg; do
    case $arg in
    # 要处理的表名
    t)
        TABLE=$OPTARG
        ;;
    # 日期
    d)
        DT=$OPTARG
        ;;
    # 环比增长指标下限
    s)
        MIN=$OPTARG
        ;;
    # 环比增长指标上限
    x)
        MAX=$OPTARG
        ;;
    # 告警级别
    l)
        LEVEL=$OPTARG
        ;;
    ?)
        echo "unkonw argument"
        exit 1
        ;;
    esac
    done

    #如果dt和level没有设置，那么默认值dt是昨天 告警级别是0
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)
    [ "$LEVEL" ] || LEVEL=0

    # 数仓DB名称
    HIVE_DB=gmall

    # 查询引擎
    HIVE_ENGINE=hive

    # MySQL相关配置
    mysql_user="root"
    mysql_passwd="000000"
    mysql_host="hadoop102"
    mysql_DB="data_supervisor"
    mysql_tbl="day_on_day"

    # 认证为hive用户，如在非安全(Hadoop未启用Kerberos认证)环境中，则无需认证
    kinit -kt /etc/security/keytab/hive.keytab hive

    # 昨日数据量
    YESTERDAY=$($HIVE_ENGINE -e "set hive.cli.print.header=false; select count(1) from $HIVE_DB.$TABLE where dt=date_add('$DT',-1);")

    # 今日数据量
    TODAY=$($HIVE_ENGINE -e "set hive.cli.print.header=false;select count(1) from $HIVE_DB.$TABLE where dt='$DT';")

    # 计算环比增长值
    if [ "$YESTERDAY" -ne 0 ]; then
    RESULT=$(awk "BEGIN{print ($TODAY-$YESTERDAY)/$YESTERDAY*100}")
    else
    RESULT=10000
    fi

    # 将结果写入MySQL表格
    mysql -h"$mysql_host" -u"$mysql_user" -p"$mysql_passwd" \
    -e"INSERT INTO $mysql_DB.$mysql_tbl VALUES('$DT', '$TABLE', $RESULT, $MIN, $MAX, $LEVEL)
    ON DUPLICATE KEY UPDATE \`value\`=$RESULT, value_min=$MIN, value_max=$MAX, notification_level=$LEVEL;"
    ```

    5. 数据量同比检查脚本(计算数据量同比增长值，并将结果和自己定义的阈值上下限，插入到MySQL表中。) week_on_week.sh

    ```shell
    #!/usr/bin/env bash
    # -*- coding: utf-8 -*-
    # 计算一张表一周数据量同比增长值
    # 参数解析
    while getopts "t:d:s:x:l:" arg; do
    case $arg in
    # 要处理的表名
    t)
        TABLE=$OPTARG
        ;;
    # 日期
    d)
        DT=$OPTARG
        ;;
    # 同比增长指标下限
    s)
        MIN=$OPTARG
        ;;
    # 同比增长指标上限
    x)
        MAX=$OPTARG
        ;;
    # 告警级别
    l)
        LEVEL=$OPTARG
        ;;
    ?)
        echo "unkonw argument"
        exit 1
        ;;
    esac
    done

    #如果dt和level没有设置，那么默认值dt是昨天 告警级别是0
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)
    [ "$LEVEL" ] || LEVEL=0

    # 数仓DB名称
    HIVE_DB=gmall

    # 查询引擎
    HIVE_ENGINE=hive

    # MySQL相关配置
    mysql_user="root"
    mysql_passwd="000000"
    mysql_host="hadoop102"
    mysql_DB="data_supervisor"
    mysql_tbl="week_on_week"

    # 认证为hive用户，如在非安全(Hadoop未启用Kerberos认证)环境中，则无需认证
    kinit -kt /etc/security/keytab/hive.keytab hive

    # 上周数据量
    LASTWEEK=$($HIVE_ENGINE -e "set hive.cli.print.header=false;select count(1) from $HIVE_DB.$TABLE where dt=date_add('$DT',-7);")

    # 本周数据量
    THISWEEK=$($HIVE_ENGINE -e "set hive.cli.print.header=false;select count(1) from $HIVE_DB.$TABLE where dt='$DT';")

    # 计算增长
    if [ $LASTWEEK -ne 0 ]; then
    RESULT=$(awk "BEGIN{print ($THISWEEK-$LASTWEEK)/$LASTWEEK*100}")
    else
    RESULT=10000
    fi

    # 将结果写入MySQL
    mysql -h"$mysql_host" -u"$mysql_user" -p"$mysql_passwd" \
    -e"INSERT INTO $mysql_DB.$mysql_tbl VALUES('$DT', '$TABLE', $RESULT, $MIN, $MAX, $LEVEL)
    ON DUPLICATE KEY UPDATE \`value\`=$RESULT, value_min=$MIN, value_max=$MAX, notification_level=$LEVEL;"
    ```

2. 数仓各层检测脚本编写

    1. ODS层 check_ods.sh

    ```shell
    #!/usr/bin/env bash
    DT=$1
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)

    #检查表 ods_order_info 数据量日环比增长
    #参数： -t 表名
    #      -d 日期
    #      -s 环比增长下限
    #      -x 环比增长上限
    #      -l 告警级别
    bash day_on_day.sh -t ods_order_info -d "$DT" -s -10 -x 10 -l 1

    #检查表 ods_order_info 数据量周同比增长
    #参数： -t 表名
    #      -d 日期
    #      -s 同比增长下限
    #      -x 同比增长上限
    #      -l 告警级别
    bash week_on_week.sh -t ods_order_info -d "$DT" -s -10 -x 50 -l 1

    #检查表 ods_order_info 订单异常值
    #参数： -t 表名
    #      -d 日期
    #      -s 指标下限
    #      -x 指标上限
    #      -l 告警级别
    #      -a 值域下限
    #      -b 值域上限
    bash range.sh -t ods_order_info -d "$DT" -c final_amount -a 0 -b 100000 -s 0 -x 100 -l 1 
    ```

    2. DWD层 check_dwd.sh

    ```shell
    #!/usr/bin/env bash
    DT=$1
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)

    # 检查表 dwd_order_info 重复ID
    #参数： -t 表名
    #      -d 日期
    #      -c 检查重复值的列
    #      -s 异常指标下限
    #      -x 异常指标上限
    #      -l 告警级别
    bash duplicate.sh -t dwd_order_info -d "$DT" -c id -s 0 -x 5 -l 0

    #检查表 dwd_order_info 的空ID
    #参数： -t 表名
    #      -d 日期
    #      -c 检查空值的列
    #      -s 异常指标下限
    #      -x 异常指标上限
    #      -l 告警级别
    bash null_id.sh -t dwd_order_info -d "$DT" -c id -s 0 -x 10 -l 0
    ```

    3. DIM层 check_dim.sh

    ```shell
    #!/usr/bin/env bash
    DT=$1
    [ "$DT" ] || DT=$(date -d '-1 day' +%F)

    #检查表 dim_user_info 的重复ID
    #参数： -t 表名
    #      -d 日期
    #      -c 检查重复值的列
    #      -s 异常指标下限
    #      -x 异常指标上限
    #      -l 告警级别
    bash duplicate.sh -t dim_user_info -d "$DT" -c id -s 0 -x 5 -l 0

    #检查表 dim_user_info 的空ID
    #参数： -t 表名
    #      -d 日期
    #      -c 检查空值的列
    #      -s 异常指标下限
    #      -x 异常指标上限
    #      -l 告警级别
    bash null_id.sh -t dim_user_info -d "$DT" -c id -s 0 -x 10 -l 0
    ```


#### 告警集成模块

1. 环境准备

    1. 在MySQL官网下载mysql-connector-python-2.1.7-1.el7.x86_64.rpm

    2. 每个节点 `rpm -i mysql-connector-python-2.1.7-1.el7.x86_64.rpm`

2. 新建python脚本用于查询数据监控结果表格并发送告警邮件 check_notification.py。（需要用到睿象云或其他第三方平台）

* read_table用于读取指标有问题的数据

* one_alert函数用于向睿象云发送告警
* mail_alert函数用于发送邮件告警

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import mysql.connector
import sys
import smtplib
from email.mime.text import MIMEText
from email.header import Header
import datetime
import urllib
import urllib2
import random


def get_yesterday():
    """
    :return: 前一天的日期
    """
    today = datetime.date.today()
    one_day = datetime.timedelta(days=1)
    yesterday = today - one_day
    return str(yesterday)


def read_table(table, dt):
    """
    :param table:读取的表名
    :param dt:读取的数据日期
    :return:表中的异常数据(统计结果超出规定上下限的数据)
    """

    # mysql必要参数设置，需根据实际情况作出修改
    mysql_user = "root"
    mysql_password = "000000"
    mysql_host = "hadoop102"
    mysql_schema = "data_supervisor"

    # 获取Mysql数据库连接
    connect = mysql.connector.connect(user=mysql_user, password=mysql_password, host=mysql_host, database=mysql_schema)
    cursor = connect.cursor()

    # 查询表头
    # ['dt', 'tbl', 'col', 'value', 'value_min', 'value_max', 'notification_level']
    query = "desc " + table
    cursor.execute(query)
    head = map(lambda x: str(x[0]), cursor.fetchall())

    # 查询异常数据(统计结果超出规定上下限的数据)
    # [(datetime.date(2021, 7, 16), u'dim_user_info', u'id', 7, 0, 5, 1),
    # (datetime.date(2021, 7, 16), u'dwd_order_id', u'id', 10, 0, 5, 1)]
    query = ("select * from " + table + " where dt='" + dt + "' and `value` not between value_min and value_max")
    cursor.execute(query)
    cursor_fetchall = cursor.fetchall()

    # 将指标和表头映射成为dict数组
    #[{'notification_level': 1, 'value_min': 0, 'value': 7, 'col': u'id', 'tbl': u'dim_user_info', 'dt': datetime.date(2021, 7, 16), 'value_max': 5},
    # {'notification_level': 1, 'value_min': 0, 'value': 10, 'col': u'id', 'tbl': u'dwd_order_id', 'dt': datetime.date(2021, 7, 16), 'value_max': 5}]
    fetchall = map(lambda x: dict(x), map(lambda x: zip(head, x), cursor_fetchall))
    return fetchall


def one_alert(line):
    """
    集成第三方告警平台睿象云，使用其提供的通知媒介发送告警信息
    :param line: 一个等待通知的异常记录，{'notification_level': 1, 'value_min': 0, 'value': 7, 'col': u'id', 'tbl': u'dim_user_info', 'dt': datetime.date(2021, 7, 16), 'value_max': 5}
    """

    # 集成睿象云需要使用的rest接口，和APP KEY，须在睿象云平台获取
    one_alert_key = "c2030c9a-7896-426f-bd64-59a8889ac8e3"
    one_alert_host = "http://api.aiops.com/alert/api/event"

    # 根据睿象云的rest api要求，传入必要的参数
    data = {
        "app": one_alert_key,
        "eventType": "trigger",
        "eventId": str(random.randint(10000, 99999)),
        "alarmName": "".join(["表格", str(line["tbl"]), "数据异常."]),
        "alarmContent": "".join(["指标", str(line["norm"]), "值为", str(line["value"]),
                                 ", 应为", str(line["value_min"]), "-", str(line["value_max"]),
                                 ", 参考信息：" + str(line["col"]) if line.get("col") else ""]),
        "priority": line["notification_level"] + 1
    }

    # 使用urllib和urllib2向睿象云的rest结构发送请求，从而触发睿象云的通知策略
    body = urllib.urlencode(data)
    request = urllib2.Request(one_alert_host, body)
    urlopen = urllib2.urlopen(request).read().decode('utf-8')
    print urlopen


def mail_alert(line):
    """
    使用电子邮件的方式发送告警信息
    :param line: 一个等待通知的异常记录，{'notification_level': 1, 'value_min': 0, 'value': 7, 'col': u'id', 'tbl': u'dim_user_info', 'dt': datetime.date(2021, 7, 16), 'value_max': 5}
    """

    # smtp协议发送邮件的必要设置
    mail_host = "smtp.126.com"
    mail_user = "skiinder@126.com"
    mail_pass = "KADEMQZWCPFWZETF"

    # 告警内容
    message = ["".join(["表格", str(line["tbl"]), "数据异常."]),
               "".join(["指标", str(line["norm"]), "值为", str(line["value"]),
                        ", 应为", str(line["value_min"]), "-", str(line["value_max"]),
                        ", 参考信息：" + str(line["col"]) if line.get("col") else ""])]
    # 告警邮件，发件人
    sender = mail_user

    # 告警邮件，收件人
    receivers = [mail_user]

    # 将邮件内容转为html格式
    mail_content = MIMEText("".join(["<html>", "<br>".join(message), "</html>"]), "html", "utf-8")
    mail_content["from"] = sender
    mail_content["to"] = receivers[0]
    mail_content["Subject"] = Header(message[0], "utf-8")

    # 使用smtplib发送邮件
    try:
        smtp = smtplib.SMTP_SSL()
        smtp.connect(mail_host, 465)
        smtp.login(mail_user, mail_pass)
        content_as_string = mail_content.as_string()
        smtp.sendmail(sender, receivers, content_as_string)
    except smtplib.SMTPException as e:
        print e


def main(argv):
    """
    :param argv: 系统参数，共三个，第一个为python脚本本身，第二个为告警方式，第三个为日期
    """
    
    # 如果没有传入日期参数，将日期定为昨天
    if len(argv) >= 3:
        dt = argv[2]
    else:
        dt = get_yesterday()

    notification_level = 0

    # 通过参数设置告警方式，默认是睿象云
    alert = None
    if len(argv) >= 2:
        alert = {
            "mail": mail_alert,
            "one": one_alert
        }[argv[1]]
    if not alert:
        alert = one_alert

    # 遍历所有表，查询所有错误内容，如果大于设定警告等级，就发送警告
    for table in ["day_on_day", "duplicate", "null_id", "rng", "week_on_week"]:
        for line in read_table(table, dt):
            if line["notification_level"] >= notification_level:
                line["norm"] = table
                alert(line)


if __name__ == "__main__":
    # 两个命令行参数
    # 第一个为警告类型：one或者mail
    # 第二个为日期，留空取昨天
    main(sys.argv)
```

#### 调度模块

1. 采用Azkaban进行调度

2. 该脚本主要是对Azkaban API的封装，主要有三个方法：azclient.py

* login函数可以登录Azkanban并返回session_id

* get_exec_id函数可以获取正在执行的工作流程的Execution ID
* wait_node可以等待指定Flow中某一结点执行完毕并判断其是否执行成功

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import time
import urllib
import urllib2
import json

# Azkaban API 接口地址
az_url = "http://hadoop102:8081/"
# Azkaban用户名
az_username = "atguigu"
# Azkaban密码
az_password = "atguigu"
# 工程名称
project = "gmall"
# flow名称
flow = "gmall"


def post(url, data):
    """
    发送post请求到指定网址

    :param url: 指定网址
    :param data: 请求参数
    :return: 请求结果
    """
    body = urllib.urlencode(data)
    request = urllib2.Request(url, body)
    urlopen = urllib2.urlopen(request).read().decode('utf-8')
    return json.loads(urlopen)


def get(url, data):
    """
    发送get请求到指定网址

    :param url: 指定网址
    :param data: 请求参数
    :return: 请求结果
    """
    body = urllib.urlencode(data)
    urlopen = urllib2.urlopen(url + body).read().decode('utf-8')
    return json.loads(urlopen)


def login():
    """
    使用`Authenticate`API进行azkaban身份认证，获取session ID

    :return: 返回session_id
    """
    data = {
        "action": "login",
        "username": az_username,
        "password": az_password
    }
    auth = post(az_url, data)
    return str(auth.get(u"session.id"))


def get_exec_id(session_id):
    """
    使用`Fetch Running Executions of a Flow`API获取正在执行的Flow的ExecId

    :param session_id: 和azkaban通讯的session_id
    :param project: 项目名称
    :param flow: 工作流名称
    :return: 执行ID
    """
    data = {
        "session.id": session_id,
        "ajax": "getRunning",
        "project": project,
        "flow": flow
    }
    execs = get(az_url + "executor?", data).get(u"execIds")
    if execs:
        return str(execs[0])
    else:
        return None


def wait_node(session_id, exec_id, node_id):
    """
    循环使用`Fetch a Flow Execution`API获取指定Flow中的某个节点(job)的执行状态，直到其执行完成

    :param session_id: 和azkaban通讯的session_id
    :param exec_id: 执行ID
    :param node_id: 指定节点(job)
    :return: 该节点是否成功执行完毕
    """
    data = {
        "session.id": session_id,
        "ajax": "fetchexecflow",
        "execid": exec_id
    }
    status = None

    # 若指定Flow中的指定Node(job)的执行状态是未完成的状态，就一直循环
    while status not in ["SUCCEEDED", "FAILED", "CANCELLED", "SKIPPED", "KILLED"]:
        # 获取指定Flow的当前的执行信息
        flow_exec = get(az_url + "executor?", data)
        # 从该Flow的执行信息中获取nodes字段的值，并遍历寻找特定的节点(job)信息，进而获取该节点(job)的状态
        for node in flow_exec.get(u"nodes"):
            if unicode(node_id) == node.get(u"id"):
                status = str(node.get(u"status"))
        print " ".join([node_id, status])
        # 等待1s，进入下一轮循环判断
        time.sleep(1)
    return status == "SUCCEEDED"
```

3. ODS层调度脚本。检查ODS层数据质量 check_ods.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys
import os
from azclient import login,wait_node,get_exec_id
from check_notification import get_yesterday

def check_ods(dt, session_id, exec_id):
    """
    检查ODS层数据质量

    :param dt: 日期
    :param session_id: 和azkaban通讯的session_id
    :param exec_id: 指定的执行ID
    :return: None
    """
    if wait_node(session_id, exec_id, "hdfs_to_ods_db") and wait_node(session_id, exec_id, "hdfs_to_ods_log"):
        os.system("bash check_ods.sh " + dt)

if __name__ == '__main__':
    argv = sys.argv
    # 获取session_id
    session_id = login()

    # 获取执行ID。只有在原Flow正在执行时才能获取
    exec_id = get_exec_id(session_id)

    # 获取日期，如果不存在取昨天
    if len(argv) >= 2:
        dt = argv[1]
    else:
        dt = get_yesterday()

    # 检查各层数据质量
    if exec_id:
        check_ods(dt, session_id, exec_id)
```

4. DWD层调度脚本。检查DWD层数据质量 check_dwd.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys
import os
from azclient import login, wait_node, get_exec_id
from check_notification import get_yesterday


def check_dwd(dt, session_id, exec_id):
    """
    检查DWD层数据质量

    :param dt: 日期
    :param session_id: 和azkaban通讯的session_id
    :param exec_id: 指定的执行ID
    :return: None
    """
    if wait_node(session_id, exec_id, "ods_to_dwd_db") and wait_node(session_id, exec_id, "ods_to_dwd_log"):
        os.system("bash check_dwd.sh " + dt)


if __name__ == '__main__':
    argv = sys.argv
    # 获取session_id
    session_id = login()

    # 获取执行ID。只有在原Flow正在执行时才能获取
    exec_id = get_exec_id(session_id)

    # 获取日期，如果不存在取昨天
    if len(argv) >= 2:
        dt = argv[1]
    else:
        dt = get_yesterday()

    # 检查各层数据质量
    if exec_id:
        check_dwd(dt, session_id, exec_id)
```

5. DIM层调度脚本。检查DIM层数据质量  check_dim.py

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys
import os
from azclient import login, wait_node, get_exec_id
from check_notification import get_yesterday


def check_dim(dt, session_id, exec_id):
    """
    检查DIM层数据质量

    :param dt: 日期
    :param session_id: 和azkaban通讯的session_id
    :param exec_id: 指定的执行ID
    :return: None
    """
    if wait_node(session_id, exec_id, "ods_to_dim_db"):
        os.system("bash check_dim.sh " + dt)


if __name__ == '__main__':
    argv = sys.argv
    # 获取session_id
    session_id = login()

    # 获取执行ID。只有在原Flow正在执行时才能获取
    exec_id = get_exec_id(session_id)

    # 获取日期，如果不存在取昨天
    if len(argv) >= 2:
        dt = argv[1]
    else:
        dt = get_yesterday()

    # 检查各层数据质量
    if exec_id:
        check_dim(dt, session_id, exec_id)
```

6. Azkaban工作流配置文件 azkaban.project,内容 `azkaban-flow-version: 2.0`；data_supervisor.flow。将所有文件打包成data_supervisor.zip文件。上传

```conf
nodes:
  - name: check_ods
    type: command
    config:
     command: python check_ods.py ${dt}

  - name: check_dwd
    type: command
    config:
     command: python check_dwd.py ${dt}

  - name: check_dim
    type: command
    config:
     command: python check_dim.py ${dt}
    
  - name: check_notification
    type: command
    dependsOn:
         - check_ods
         - check_dwd
         - check_dim
    config:
     command: python check_notification.py ${alert} ${dt}
```

### 可视化模块

1. 采用Superset进行可视化展示

2. 在Superset中新建数据库连接

3. dataset加入全部表格

4. 新建一个dashboard，新建表，配置表，把表加入dashboard中

## Solr简介

0. solr是基于Lucence开发的企业级搜索引擎技术，而lucence的原理是倒排索引。

1. 安装Solr-7.7.3

    1. 在每台节点创建系统用户solr

    2. 解压solr-7.74.tgz 到指定目录，并改名为solr

    3. 修改solr目录所有者为solr

    4. 修改solr配置文件 ${SOLR_HOME}/bin/solr.in.sh

    ```shell
    ZK_HOST="hadoop102:2181,hadoop103:2181,hadoop104:2181"
    ```

    5. 分发

    6. 启动zookeeper

    7. 使用solr用户启动solr（所有节点）`sudo -i -u solr /opt/module/solr/bin/solr start`

    8. 修改打开文件数限制 vim /etc/security/limits.conf 增加(选做)；修改进程数限制 vim /etc/security/limits.d/20-nproc.conf 。solr重启服务器

    ```conf
    * soft nofile 65000
    * hard nofile 65000
    ```

    ```conf
    *          soft    nproc     65000
    ```

    9. 访问web页面 http://hadoop102:8983

2. 添加文档

    * 把solrJ的jar包添加到工程中。

    * 创建一个SolrServer，使用HttpSolrServer创建对象。

    * 创建一个文档对象SolrInputDocument对象。

    * 向文档中添加域。必须有id域，域的名称必须在schema.xml中定义。（solrhome/core/conf文件下面。solr webapp 的classload下面找到他.）

    * 把文档添加到索引库中。

    * 提交。

3. 其他操作查询文档

