# 概述

本次主体为**influxDB**，简述influxDB的市场情况。

# 目的

## 1. 了解事件序列数据库

​	时间序列数据库（TSDB）是针对带时间戳或时间序列数据优化的数据库。时间序列只是随时间跟踪，监控，下采样和聚合的度量或事件。这可以是服务器指标，应用程序性能监控，网络数据，传感器数据，事件，点击，市场交易以及许多其他类型的分析数据。

​	随着系统监控以及物联网的发展，已经开始受到更多的关注。 维基百科上对于时间序列的定义是‘一系列数据点按照时间顺序排列’， 但是我个人的理解是**存储在服务端的客户端历史**。 时间序列数据就是历史，它具有**不变性**, **唯一性**以及**可排序性**。 并且**数据到达服务器的顺序并不影响正确性**，根据数据本身可以直接进行排序和去重。

## 2. 时序数据库的优势

### 1. 低冗余

​	同一数据源的tags不再冗余存储。一个Block内的数据都共用一个SeriesKey，只需要将这个SeriesKey写入这个Block的Trailer部分就可以。大大降低了时序数据的存储量。

### 2. 独立储存

​	时间序列和value可以在同一个Block内分开独立存储，独立存储就可以对时间列以及数值列分别进行压缩。InfluxDB对时间列的存储借鉴了Beringei的压缩方式，使用delta-delta压缩方式极大的提高了压缩效率。而对Value的压缩可以针对不同的数据类型采用相同的压缩效率。

### 3. 高效

​	对于给定数据源以及时间范围的数据查找，可以非常高效的进行查找。

## 3. 传统数据库的劣势

### 1. 高并发和大规模

​	时间序列数据累计速度非常快。（例如，一辆联网汽车[每小时能收集25G），常规数据库在设计之初并非处理这种规模的数据，关系型数据库处理大数据集的效果非常糟糕。

### 2. 可用性

​	TSDB通常还包括一些共通的对时间序列数据分析的功能和操作：数据保留策略、连续查询、灵活的时间聚合等。即使当下不考虑规模（例如，您刚开始收集数据），这些功能仍可提供更好的用户体验，使你的生活更轻松。

# 主要方式

### 1. github

### 2. API文档

### 3. API测试

### 4. 论坛介绍

# 主要内容

## 1. 运行influxdb

​	此次测试使用docker 镜像进行测试，未做实际部署。

### 1. 获取influxdb 的docekr镜像

> docker pull influxdb

​	容器运行：

```
docker run -p 8086:8086 -v $PWD:/var/lib/influxdb  influxdb
```

### 2. 操作

#### 1. 使用HTTP API操作

​	执行查询，`GET`请向`/query`端点发送请求，将URL参数设置`db`为目标数据库，并将URL参数设置`q`为查询。您也可以`POST`通过发送相同参数作为URL参数或作为正文的一部分来使用请求`application/x-www-form-urlencoded`：

```shell
curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mydb" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
```

​	InfluxDB返回JSON, 查询结果显示在`"results"`数组中。在单个API调用中向InfluxDB发送多个查询。只需使用分号分隔每个查询。

​	InfluxDB支持多种查询：时间戳格式， 最大行限制。分块等。

​	创建数据库，`POST`请向`/query`端点发送请求并将URL参数设置`q`为`CREATE DATABASE <new_database_name>`。下面的示例向运行的InfluxDB发送请求`localhost`并创建数据库`mydb`：

```shell
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE mydb"
```

​	InfluxDB 同时支持多种编写数据：同时写入多个点， 从文件中写入等。

​	具体内容请参考官网[InfluxDB HTTP API 参考](https://docs.influxdata.com/influxdb/v1.6/tools/api/)。



#### 2. InfluxDB-python 操作

​	InfluxDB-Python是一个与[InfluxDB](https://influxdata.com/time-series-platform/influxdb/)交互的客户端。

​	要连接到InfluxDB，您必须创建一个 [`InfluxDBClient`](https://influxdb-python.readthedocs.io/en/latest/api-documentation.html#influxdb.InfluxDBClient)对象。默认配置`localhost`使用默认端口连接到InfluxDB。

```
from influxdb import InfluxDBClient

# using Http
client = InfluxDBClient(database='dbname')
client = InfluxDBClient(host='127.0.0.1', port=8086, database='dbname')
client = InfluxDBClient(host='127.0.0.1', port=8086, username='root', password='root', database='dbname')
```

​	要编写pandas DataFrames或将数据读入pandas DataFrame，请使用[`DataFrameClient`](https://influxdb-python.readthedocs.io/en/latest/api-documentation.html#influxdb.DataFrameClient)对象。这些客户端的启动方式与 [`InfluxDBClient`](https://influxdb-python.readthedocs.io/en/latest/api-documentation.html#influxdb.InfluxDBClient)：

```
from influxdb import DataFrameClient

client = DataFrameClient(host='127.0.0.1', port=8086, username='root', password='root', database='dbname')
```

​	以下是使用InfluxDB-python案例：

```
user = 'root'
password = 'root'
dbname = 'example'
dbuser = 'smly'
dbuser_password = 'my_secret_password'
query = 'select value from cpu_load_short;'
json_body = [
    {
        "measurement": "cpu_load_short",
        "tags": {
            "host": "server01",
            "region": "us-west"
        },
        "time": "2009-11-10T23:00:00Z",
        "fields": {
            "Float_value": 0.64,
            "Int_value": 3,
            "String_value": "Text",
            "Bool_value": True
        }
    }
]

client = InfluxDBClient(host, port, user, password, dbname)

print("创建数据库: " + dbname)
client.create_database(dbname)

print("创建保留策略")
client.create_retention_policy('awesome_policy', '3d', 3, default=True)

print("切换用户: " + dbuser)
client.switch_user(dbuser, dbuser_password)
print("写入: {0}".format(json_body))
client.write_points(json_body)

print("查询数据: " + query)
result = client.query(query)
print("Result: {0}".format(result))

print("切换用户: " + user)
client.switch_user(user, password)

print("删除数据库: " + dbname)
client.drop_database(dbname)
```

​	由此看出InfluxDB 操作简单。易于学习。 	并且InfluxDB支持R， Ruby， Java， Javascripy/node.js，MATLAB，Perl，PHP等多种主流语言。详情请参考[InfluxDB API客户端库](https://docs.influxdata.com/influxdb/v1.6/tools/api_client_libraries/)。



#### 3. InfluxDB命令行操作（CLI/shell）

​	InfluxDB的命令行界面（`influx`）是HTTP API的交互式shell。使用`influx`写在不同格式的数据（手动或从文件中），交互查询数据，和视图查询输出。

​	要访问CLI，首先启动`influxd`数据库进程，然后`influx`在终端中启动。进入shell并成功连接到InfluxDB节点。并且`influx`启动时可以传递几个参数：

```
-compressed 如果导入文件已压缩，则设置为true。使用-import。

-consistency 'any|one|quorum|all' 设置写入一致性级别。

-database 'database name'influx连接 的数据库。

-execute 'command' 执行InfluxQL命令并退出
```

此文档只列出一部分， 详情请参考[InfluxDB的命令行操作](https://docs.influxdata.com/influxdb/v1.6/tools/shell/#influx-arguments)。

# 结果

​	InfluxDB在单个节点上速度非常快，并且具有许多非常多的功能和高效的存储空间。InfluxDB使用一种易于入门的SQL样式查询语言。查询语法简单，功能强。后端时序数据操作、写入块。

​	简化的[冲突解决方案可](https://docs.influxdata.com/influxdb/v1.6/troubleshooting/frequently-asked-questions/#how-does-influxdb-handle-duplicate-points)提高写入性能。无法存储重复数据; 可能会在极少数情况下覆盖数据。

​	限制对删除的访问可以提高查询和写入性能。删除功能受到严重限制。

​	限制对更新的访问可以提高查询和写入性能。更新功能受到严重限制。

​	绝大多数写入用于具有最新时间戳的数据，并且数据按时间升序添加。按时间升序添加数据的性能要高得多。

​	规模至关重要。该数据库必须能够处理*大*容量的读取和写入。可以处理*大*容量的读取和写入。如果数据库负载很重，查询返回可能不包括最新的点。

​	能够编写和查询数据比拥有强一致的视图。编写和查询数据库可以由多个客户端在高负载下完成。

​	许多时间[序列](https://docs.influxdata.com/influxdb/v1.6/concepts/glossary/#series)都是短暂的。通常情况下，时间序列只出现几个小时然后消失。InfluxDB擅长管理不连续数据。无架构设计意味着不支持某些数据库功能，例如没有交叉表连接。

​	InfluxDB有非常强大的工具来处理聚合数据和大数据集。点没有传统意义上的ID，它们按时间戳和系列区分。

**大规模的可扩展性和性能：**有效的时间序列数据库使应用程序可以轻松扩展，以便在连续流中支持数百万个物联网设备或时间序列数据点，并执行实时分析。

**减少停机时间：**在停机时间不可接受的情况下，为时间序列数据构建的数据库体系结构可确保即使在网络分区或硬件故障时数据始终可用。

**降低成本：**高弹性转化为管理中断所需的资源更少。使用商用硬件快速轻松地进行扩展可降低扩展或缩小的操作和硬件成本。

**改进的业务决策：**通过使组织能够实时分析数据，时间序列数据库可帮助组织更快，更准确地调整能耗，设备维护，基础架构变更或影响业务的其他重要决策。

