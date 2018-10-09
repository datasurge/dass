## Influxdb 使用记录

### 数据库管理

设置时间格式为可视化格式：precision rfc3339， 或者进入shell时使用 -precision rfc3339。

#### 创建数据库

```
CREATE DATABASE <database_name> [WITH [DURATION <duration>] [REPLICATION <n>] [SHARD DURATION <duration>] [NAME <retention-policy-name>]]
```

CREATE DATABASE需要一个数据库名称。

`WITH`，`DURATION`，`REPLICATION`，`SHARD DURATION`和`NAME`子句是可选的用来创建与数据库相关联的单个保留策略。如果您没有在`WITH`之后指定其中一个子句，将默认为`autogen`保留策略。创建的保留策略将自动用作数据库的默认保留策略。

一个成功的`CREATE DATABASE`查询返回一个空的结果。如果您尝试创建已存在的数据库，InfluxDB什么都不做，也不会返回错误。

##### 1. 创建数据库

```
CREATE DATABASE "NOAA_water_database"
```

该语句创建了一个叫做`NOAA_water_database`的数据库，默认InfluxDB也会创建`autogen`保留策略，并和数据库`NOAA_water_database`关联起来。

##### 2. 创建有保留策略的数据库

```
CREATE DATABASE "NOAA_water_database" WITH DURATION 3d REPLICATION 1 SHARD DURATION 1h NAME "liquid"
```

该语句创建了一个叫做`NOAA_water_database`的数据库，并且创建了`liquid`作为数据库的默认保留策略，其持续时间为3天，副本数是1，shard group的持续时间为一个小时。

#### 删除数据库

```
DROP DATABASE <database_name>
```

`DROP DATABASE`从指定数据库删除所有的数据，以及measurement，series，continuous queries, 和retention policies

##### 使用drop从索引中删除series

`ROP SERIES`删除一个数据库里的一个series的所有数据，并且从索引中删除series。

> `DROP SERIES`不支持`WHERE`中带时间间隔。

该查询采用以下形式，您必须指定`FROM`子句或`WHERE`子句：

```
DROP SERIES FROM <measurement_name[,measurement_name]> WHERE <tag_key>='<tag_value>'
```

从单个measurement删除所有series：

```
DROP SERIES FROM "h2o_feet"
```

从单个measurement删除指定tag的series：

```
DROP SERIES FROM "h2o_feet" WHERE "location" = 'santa_monica'
```

从数据库删除有指定tag的所有measurement中的所有数据：

```
DROP SERIES WHERE "location" = 'santa_monica'
```

#### 使用delete 删除series

`DELETE`删除数据库中的measurement中的所有点。与`DROP SERIES`不同，它不会从索引中删除series，并且它支持`WHERE`子句中的时间间隔。

该查询采用以下格式，必须包含`FROM`子句或`WHERE`子句，或两者都有：

```
DELETE FROM <measurement_name> WHERE [<tag_key>='<tag_value>'] | [<time interval>]
```

删除measurement`h2o_feet`的所有相关数据：

```
DELETE FROM "h2o_feet"
```

删除measurement`h2o_quality`并且tag`randtag`等于3的所有数据：

```
DELETE FROM "h2o_quality" WHERE "randtag" = '3'
```

删除数据库中2016年一月一号之前的所有数据：

```
DELETE WHERE time < '2016-01-01
```

- 当指定measurement名称时，`DELETE`在`FROM子`句中支持正则表达式，并在指定tag时支持`WHERE`子句中的正则表达式。
- `DELETE`不支持`WHERE`子句中的field。
- 如果你需要删除之后的数据点，则必须指定`DELETE SERIES`的时间间隔，因为其默认运行的时间为`time <now()`

#### 删除measurement

`DROP MEASUREMENT`删除指定measurement的所有数据和series，并且从索引中删除measurement。

该语法格式为：

```
DROP MEASUREMENT <measurement_name>
```

注意：`DROP MEASUREMENT`删除measurement中的所有数据和series，但是不会删除相关的continuous queries。

#### 删除shard

`DORP SHARD`删除一个shard，也会从metastore中删除shard。格式如下：

```
DROP SHARD <shard_id_number>
```

#### 保留策略

以下部分介绍如何创建，更改和删除保留策略。 请注意，创建数据库时，InfluxDB会自动创建一个名为`autogen`的保留策略，该保留策略保留时间为无限。您可以重命名该保留策略或在配置文件中禁用其自动创建。

```
CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> [SHARD DURATION <duration>] [DEFAULT]
```

`DURATION`子句确定InfluxDB保留数据的时间。 `<duration>`是持续时间字符串或INF（无限）。 保留策略的最短持续时间为1小时，最大持续时间为INF。

`REPLICATION`子句确定每个点的多少独立副本存储在集群中，其中`n`是数据节点的数量。该子句不能用于单节点实例。

`SHARD DURATION`子句确定shard group覆盖的时间范围。 `<duration>`是一个持续时间字符串，不支持INF（无限）持续时间。此设置是可选的。 默认情况下，shard group持续时间由保留策略的`DURATION`决定：

| 保留策略的持续时间 | shard group的持续时间 |
| ------------------ | --------------------- |
| < 2天              | 1小时                 |
| >= 2天并<=6个月    | 1天                   |
| > 6个月            | 7天                   |

最小允许`SHARD GROUP DURATION`为1小时。 如果`CREATE RETENTION POLICY`查询尝试将`SHARD GROUP DURATION`设置为小于1小时且大于0，则InfluxDB会自动将`SHARD GROUP DURATION`设置为1h。 如果`CREATE RETENTION POLICY`查询尝试将`SHARD GROUP DURATION`设置为0，InfluxDB会根据上面列出的默认设置自动设置`SHARD GROUP DURATION`。

`DEFAULT`将新的保留策略设置为数据库的默认保留策略。此设置是可选的。

##### 1. 创建

```
create retention policy "one_day_only" on "NOAA_water_database" duration 1d replication 1
```

给数据库`NOAA_water_database`创建一个保留策略`one_day_only`，持续时间为1天，副本数为1.

##### 2. 创建默认的

```
 create retention policy "one_day_only" on "NOAA_water_database" duration 23h60m replication 1 default
```

创建与上面示例中相同的保留策略，但将其设置为数据库的默认保留策略。

成功的`CREATE RETENTION POLICY`执行返回为空。如果您尝试创建与已存在的保留策略相同的保留策略，InfluxDB不会返回错误。 如果您尝试创建与现有保留策略名称相同但具有不同属性的保留策略，InfluxDB会返回错误。

可以再数据库是指定一个新的保留策略。

#### 修改保留策略

`ALTER RETENTION POLICY`形式如下，你必须至少指定一个属性：`DURATION`, `REPLICATION`, `SHARD DURATION`,或者`DEFAULT`:

```
ALTER RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> SHARD DURATION <duration> DEFAULT
```

##### 1. 例

创建一个为期两天的保留策略

```
create retention policy "what_is_time" on "NOAA_water_database" duration 2d replication 1
```

修改`what_is_time`的持续时间为3个星期，shard group的持续时间为30分钟，并将其作为数据库`NOAA_water_database`的默认保留策略：

```
alter retention policy "what_is_time" on "NOAA_water_database" duration 3w shard duration 30m default
```

在这个例子中，`what_is_time`将保留其原始副本数为1。

#### 删除保留策略

删除指定的保留策略的所有measurement和数据：

```
DROP RETENTION POLICY <retention_policy_name> ON <database_name>
```

成功的`DROP RETENTION POLICY`返回一个空的结果。如果您尝试删除不存在的保留策略，InfluxDB不会返回错误。



### 行协议（写入协议）

InfluxDB的行协议是一种写入数据点到InfluxDB的文本格式。必须要是这样的格式的数据点才能被Influxdb解析和写入成功。

一行Line Protocol表示InfluxDB中的一个数据点。它向InfluxDB通知点的measurement，tag set，field set和timestamp。以下代码块显示了行协议的示例，并将其分解为其各个组件：

```
weather,location=us-midwest temperature=82 1465839830100400200
+-----------+--------+-+---------+-+---------+
|measurement|,tag_set| |field_set| |timestamp|
+-----------+--------+-+---------+-+---------+

```

#### measurement 

写入数据的measurement， 在行协议中是必须的， 上述的measurement为weather。

#### tag set

数据点中包含的tag， tag在行协议中是可选的。**注意**measurement和tag set使用不带空格的逗号分开的。

```
<tag key>=<tag value>
```

多组tag直接使用不带空格的逗号分开：

```
<tag key>=<tag value>,<tag key>=<tag value>
```

例如上面的tag set由一个tag组成`location=us-midwest`，现在加另一个tag(`season=summer`)，就变成了这样：

```
weather,location=us-midwest,season=summer temperature=82 1465839830100400200
```

**为了获得最佳性能，您应该在将它们发送到数据库之前按键进行排序。排序应该与[Go bytes.Compare function](http://golang.org/pkg/bytes/#Compare)的结果相匹配。**

#### 空格

分离measurement和field set，或者如果您使用数据点包含tag set，则使用空格分隔tag set和field set。行协议中空格是必需的。

没有tag set的有效行协议：

```
weather temperature=82 1465839830100400200
```

##### field set

 每个数据点在行协议中至少需要一个field。使用无空格的`=`分隔field的键值对：

```
<field_key>=<field_value>
```

多组field直接用不带空格的逗号分开：

```
<field_key>=<field_value>,<field_key>=<field_value>
```

例如上面的field set由一个field组成`temperature=82`，现在加另一个field(`bug_concentration=98`)，就变成了这样：

```
weather,location=us-midwest temperature=82,bug_concentration=98 1465839830100400200
```

#### 空格

使用空格分隔field set和可选的时间戳。如果你包含时间戳，则行协议中需要空格。

##### timestamp

数据点的时间戳记以纳秒精度Unix时间。行协议中的时间戳是可选的。 如果没有为数据点指定时间戳，InfluxDB会使用服务器的本地纳秒时间戳。

在这个例子中，时间戳记是`1465839830100400200`（这就是RFC6393格式的`2016-06-13T17：43：50.1004002Z`）。下面的行协议是相同的数据点，但没有时间戳。当InfluxDB将其写入数据库时，它将使用您的服务器的本地时间戳而不是`2016-06-13T17：43：50.1004002Z`。

```
weather,location=us-midwest temperature=82
```

官方建议使用最粗糙时间戳， 有利于压缩率。

使用网络时间协议（NTP）来同步主机之间的时间。InfluxDB使用主机在UTC的本地时间为数据分配时间戳; 如果主机的时钟与NTP不同步，写入InfluxDB的数据的时间戳可能不正确。

#### 数据类型

行协议的主要组件的数据类型：measurement，tag keys，tag values，field keys，field values和timestamp。

其中measurement，tag keys，tag values，field keys始终是字符串。

**注意：**因为InfluxDB将tag value存储为字符串，所以InfluxDB无法对tag value进行数学运算。此外，InfluxQL函数不接受tag value作为主要参数。 在设计架构时要考虑到这些信息。

Field value可以是整数、浮点数、字符串和布尔值：

- 浮点数 —— 默认是浮点数，InfluxDB假定收到的所有field value都是浮点数。
  以浮点类型存储上面的`82`：

  ```
  weather,location=us-midwest temperature=82 1465839830100400200
  ```

- 整数 —— 添加一个`i`在field之后，告诉InfluxDB以整数类型存储：
  以整数类型存储上面的`82`：

  ```
  weather,location=us-midwest temperature=82i 1465839830100400200
  ```

- 字符串 —— 双引号把字段值引起来表示字符串:
  以字符串类型存储值`too warm`：

  ```
  weather,location=us-midwest temperature="too warm" 1465839830100400200
  ```

- 布尔型 —— 表示TRUE可以用`t`,`T`,`true`,`True`,`TRUE`;表示FALSE可以用`f`,`F`,`false`,`False`或者`FALSE`：
  以布尔类型存储值`true`：

  ```
  weather,location=us-midwest too_hot=true 1465839830100400200
  ```

**注意：**数据写入和数据查询可接受的布尔语法不同。

在measurement中，field value的类型在分片内不会有差异，但在分片之间可能会有所不同。例如，如果InfluxDB尝试将整数写入到与浮点数相同的分片中，则写入会失败：

```
INSERT weather,location=us-midwest temperature=82 1465839830100400200
INSERT weather,location=us-midwest temperature=81i 1465839830100400300
# ERR: {"error":"field type conflict: input field \"temperature\" on 
# measurement \"weather\" is type int64, already exists as type float"}
```

但是，如果InfluxDB将整数写入到一个新的shard中，虽然之前写的是浮点数，那依然可以写成功：

```
INSERT weather,location=us-midwest temperature=82 1465839830100400200
INSERT weather,location=us-midwest temperature=81i 1467154750000000000
```

#### 引号

在行协议中使用双（`“`）或单（`'`）引号。

- 时间戳不要双或单引号。下面这是无效的行协议。
- field value不要单引号,即时是字符串类型。
- measurement名称，tag keys，tag value和field key不用单双引号。InfluxDB会假定引号是名称的一部分。
- 当field value是整数，浮点数或是布尔型时，不要使用双引号，不然InfluxDB会假定值是字符串类型。
- 当Field value是字符串时，使用双引号：

#### 特殊字符和关键字

##### 特殊字符

对于tag key，tag value和field key，始终使用反斜杠字符\来进行转义：

- 逗号`,`：

```
weather,location=us\,midwest temperature=82 1465839830100400200
```

- 等号`=`：

```
weather,location=us-midwest temp\=rature=82 1465839830100400200
```

- 空格：

```
weather,location\ place=us-midwest temperature=82 1465839830100400200
```

对于measurement，也要反斜杠\来转义。

- 逗号`,`：

```
wea\,ther,location=us-midwest temperature=82 1465839830100400200
```

- 空格：

```
wea\ ther,location=us-midwest temperature=82 1465839830100400200
```

字符串类型的field value，也要反斜杠\来转义。

- 双引号`"`：

```
weather,location=us-midwest temperature="too\"hot\"" 1465839830100400200
```

**注意**：行协议不要求用户转义反斜杠字符\，所有其他特殊字符也不要转义。Influxdb处理emojis也没有问题。

##### 关键字

行协议接受InfluxQL关键字作为标识符名称。一般来说，我们建议避免在schema中使用InfluxQL关键字，因为它可能会在查询数据时引起混淆。

关键字`time`是特殊情况。`time`可以是cq的名称，数据库名称，measurement名称，RP名称，subscription名称和用户名 在这种情况下，查询`time`不需要双引号。`time`不能是field key或tag key; 当把`time`作为field key或是tag key写入时，InfluxDB会拒绝并返回错误。

#### 写数据的方法

现在你知道所有关于行协议的信息，你如何在使用中用行协议写入数据到InfluxDB呢？在这里，我们将给出两个快速示例，然后可以到[工具](https://jasper-zhang1.gitbooks.io/influxdb/content/Write_protocols/line_protocol.html)部分以获取更多信息。

#### HTTP API

使用HTTP API将数据写入InfluxDB。 向`/write`端点发送`POST`请求，并在请求主体中提供您的行协议：

```
curl -i -XPOST "http://localhost:8086/write?db=science_is_cool" --data-binary 'weather,location=us-midwest temperature=82 1465839830100400200'
```

有关查询字符串参数，状态码，响应和更多示例的深入描述，请参阅[API参考](https://jasper-zhang1.gitbooks.io/influxdb/content/Write_protocols/line_protocol.html)。

#### CLI

使用InfluxDB的命令行界面（CLI）将数据写入InfluxDB。启动CLI，使用相关数据库，并将`INSERT`放在行协议之前：

```
INSERT weather,location=us-midwest temperature=82 1465839830100400200
```

你还可以使用CLI从文件导入行协议。

还有几种将数据写入InfluxDB的方式。有关HTTP API，CLI和可用的服务插件（UDP，Graphite，CollectD和OpenTSDB）的更多信息，请参阅[工具](https://jasper-zhang1.gitbooks.io/influxdb/content/Write_protocols/line_protocol.html)部分。

### 重复数据

一个点由measurement名称，tag set和timestamp唯一标识。如果您提交具有相同measurement，tag set和timestamp，但具有不同field set的行协议，则field set将变为旧field set与新field set的合并，并且如果有任何冲突以新field set为准。





### 查询语法

首先下载示例数据：

```
curl https://s3.amazonaws.com/noaa.water-database/NOAA_data.txt -o NOAA_data.txt
```

创建数据库及其导入：

```
# 类似sql 语句
create database NOAA_water_database;
# 通过CLI将数据写入influxdb: # 如果是docker容器启动， 可以先将数据移入容器在操作。
influx -import -path=NOAA_data.txt -precision=s -database=NOAA_water_database
```

#### 从单个measurement 查询语法（基本查询）

```
SELECT <field_key>[,<field_key>,<tag_key>] FROM <measurement_name>[,<measurement_name>]
```

##### 1. 查询所有的field和tag

```
select * from "h2o_feet"
```

并支持手动标注其表示符的类型, 语法是在field和tag之后使用**双冒号**分割，并写上类型：

```
 SELECT *::field FROM "h2o_feet"
 # 获取所有的field， 使用类型分割。
```

如果您使用CLI，请确保在运行查询之前输入`USE NOAA_water_database`, 需要注意**各语言连接**同样在需要输入以上步骤。

##### 2. 查询特定的tag和field

```
select 'level description', 'location', 'water_level' from 'h2o_feet';
# 类似于sql中的字段名查询。
```

并支持手动标注其表示符的类型, 语法是在field和tag之后使用**双冒号**分割，并写上类型：

```
select "level description"::field, "localtion"::tag, "water_level"::field from "h2o_feet";
```

##### 3. 选中特定的field并执行计算

```
select ("water_level" * 2) + 4 from "h2o_feet"
```

该查询将`water_level`字段值乘以2，并加上4。请注意，InfluxDB遵循标准操作顺序。

##### 4. 从多个measurement 中查询

```
select * from "h2o_feet", "h2o_pH"
# 此不是measurement 之间的连接， 只是将多个measurement数据返回。
```

##### 5. 完全限定查询

```
select * from “NOAA_water_database"."autogen"."h2o_feet"
# 查询选择数据库NOAA_water_database中的数据，autogen为存储策略，h2o_feet为
# measurement
# 使用的为点连接，
```

##### 6. 从特定数据库中查询measurement的所有数据

```
select * from "NOAA_water_database".."h2o_feet"
# 数据库名..measurement name
```



**注意点**

**一个查询在`SELECT`子句中至少需要一个field key来返回数据。如果`SELECT`子句仅包含单个tag key或多个tag key，则查询返回一个空的结果。**



#### WHERE语句

`WHERE`子句用作field，tag和timestamp的过滤。

```
SELECT_clause FROM_clause WHERE <conditional_expression> [(AND|OR) <conditional_expression> [...]]
```

**fields** `WHERE`子句支持field value是字符串，布尔型，浮点数和整数这些类型。在`WHERE`子句中单引号来表示字符串字段值。具有无引号字符串字段值或双引号字符串字段值的查询将不会返回任何数据，并且在大多数情况下也不会返回错误。

```
field_key <operator> ['string' | boolean | float | integer]
```

支持的操作符：

`=` 等于
`<>` 不等于
`!=` 不等于
`>` 大于
`>=` 大于等于
`<` 小于
`<=` 小于等于

**tags** `WHERE`子句中的用单引号来把tag value引起来。具有未用单引号的tag或双引号的tag查询将不会返回任何数据，并且在大多数情况下不会返回错误。

```
tag_key <operator> ['tag_value']
```

支持的操作符：

`=` 等于
`<>` 不等于
`!=` 不等于

**timestamps** 对于大多数`SELECT`语句，默认时间范围为UTC的`1677-09-21 00：12：43.145224194`到`2262-04-11T23：47：16.854775806Z`。 对于只有`GROUP BY time()`子句的`SELECT`语句，默认时间范围在UTC的`1677-09-21 00：12：43.145224194`和`now()`之间。

##### 1. 查询特定field的key value的数据

```
select * from "h2o_feet" where "water_level" > 8
# 查询字段water_level 大于 8 的数据 
```

##### 2. 查询特定field的key value为字符串的数据

```
select * from "h2o_feet" where "level description" = 'below 3 feet'
# 查询level description 字段等于 below 3 feet。
```

注意点 **字符串** 需要使用单引号将field value 引起来。需要注意的为**各语言**连接查询的使用引号的使用。

##### 3. 查询特定的field的key value并带有计算的数据

```
select * from "h2o_feet" where "water_level" + 2 > 11.9
# 查询字段water_level 字段加上2 并大于11.9
```

##### 4. 查询有特定的tag的key value以及特定的field的key value的数据

```
select "water_level" from "h2o_feet" where "location" <> 'santa_monica' and (water_level < -0.59 or water_level > 9.95)
#查询从h2o_feet中返回数据，其中tag location设置为santa_monica，并且field water_level的值小于-0.59或大于9.95。 WHERE子句支持运算符AND和OR，并支持用括号分隔逻辑。
# 再次提示 此处的 <> 等于 != 。 <> 为不等于的意思。
```

##### 5. 根据时间戳来过滤数据

```
select * from "h2o_feet" where time > now() - 7d
# 该查询返回measurement在过去其他的数据， 由于示例数据最后问 2015-9月， 可以使用长天数， 例：1200d(天),1200m(分钟)。 但是同样的年月写法， 错误写法：120M， 10Y。
```



**where注意点：tag value或field value缺少单引号的结果。具有无引号或双引号tag value或field value的查询将不会返回任何数据，并且在大多数情况下不会返回错误。**

结果为注意使用单引号， 尤其是各语言连接的时候输入的查询语句。



#### GROUP BY 查询

`GROUP BY`子句后面可以跟用户指定的tags或者是一个时间间隔。

>  说明：在InfluxDB中，epoch 0(`1970-01-01T00:00:00Z`)通常用作等效的空时间戳。如果要求查询不返回时间戳，例如无限时间范围的聚合函数，InfluxDB将返回epoch 0作为时间戳。

##### 1. group by tags

`group by <tag>`后面跟着用户指定的tags

```
SELECT_clause FROM_clause [WHERE_clause] GROUP BY [* | <tag_key>[,<tag_key]]
```

`GROUP BY *`
对结果中的所有tag作group by。

`GROUP BY <tag_key>`
对结果按指定的tag作group by。

`GROUP BY <tag_key>,<tag_key>`
对结果数据按多个tag作group by，其中tag key的顺序没所谓。

###### 1.单个tag分组

```
select mean("water_level") from "h2o_feet" group by "location"
```

根据字段`location`字段分组， 并进行对`water_level`字段平均值的计算。

###### 2. 对多个tag做分组

```
select mean("index") from "h2o_quality" group by "location", "randtag"
```

根据`location`和`randtag`字段分组， 并进行对`index`字段平均值的计算。多个字段进行分组，每个字段使用逗号隔开。

**个人理解**， 多个tag为合并分组，将多个字段何在一起，进行分组， 并分出字段之间所有的可能性组。为尽可能将所有的分组全部列出来。尽可能多的分组。

###### 3. 对所有的tag进行分组

```
select mean("index") from "h2o_quality" group by *
```

对measurement中所有的tag进行分组，并进行平局值计算。

##### 2. group by 时间间隔

group by time() 返回结果按指定的时间间隔分组。

```
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>),[tag_key] [fill(<fill_option>)]
```

基本`GROUP BY time()`查询需要`SELECT`子句中的InfluxQL函数和`WHERE`子句中的时间范围。请注意，`GROUP BY`子句必须在`WHERE`子句之后。

`time(time_interval)`  `GROUP BY time()`语句中的`time_interval`是一个时间duration。决定了InfluxDB按什么时间间隔group by。

`fill(<fill_option>)` `fill（<fill_option>）`是可选的。它会更改不含数据的时间间隔的返回值。

示例：

```
select "water_level", "location" from "h2o_feet" where time >= '2015-08-18T00:30:00Z' and time <= '2015-08-18T00:30:00Z'
```



###### 1. 示例以12分钟为时间间隔的group by

```
select count("water_level") from "h2o_feet" where "location"='coyote_creek' and time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:30:00Z' group by time(12)
```

该查询使用count来计算`location=coyote_creek`的`water_level`个数，并将其分组结果分为12分钟间隔。每个时间戳的结果代表一个12分钟的间隔。

###### 2. 示例以十二分钟为时间间隔， 并对tag key 做分组

```
select count("water_level") from "h2o_feet" where time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:30:00Z' group by time(12m), "location"
```

该示例计算`water_leval`的数量。它将结果按`location`分组并分隔12分钟。请注意，时间间隔和tag key在GROUP BY子句中以逗号分隔。查询返回两个measurement的结果：针对tag `location`的每个值。

##### 注意点：

使用时间分组的时间边界问题， InfluxDB依赖于`GROUP BY time()`间隔和系统预设时间边界来确定每个时间间隔中包含的原始数据以及查询返回的时间戳。

InfluxDB使用独立于`WHERE`子句中任何时间条件的`GROUP BY`间隔的预设的四舍五入时间边界。当计算结果时，所有返回的数据必须在查询的显式时间范围内发生，但`GROUP BY`间隔将基于预设的时间边界。

##### 3. 高级group by time()  使用

```
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>,<offset_interval>),[tag_key] [fill(<fill_option>)]
```

高级`GROUP BY time()`查询需要`SELECT`子句中的InfluxQL函数和`WHERE`子句中的时间范围。 请注意，`GROUP BY`子句必须在`WHERE`子句之后。

**`time(time_interval,offset_interval)`** `offset_interval`是一个持续时间。它向前或向后移动InfluxDB的预设时间界限。`offset_interval`可以为正或负。

**`fill(<fill_option>)`**  `fill(<fill_option>)`是可选的。它会更改不含数据的时间间隔的返回值。

**范围** 高级`GROUP BY time()`查询依赖于`time_interval`，`offset_interval`和InfluxDB的预设时间边界，以确定每个时间间隔中包含的原始数据以及查询返回的时间戳。

###### 1. 示例一

```
select "water_level" from "h2o_feet" where "location"='coyote_creek' and time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:30:00Z'
```

此为查询`location`等于coyote_creek, 并在时间范围之内。

###### 2. 以18分钟时间间隔分组， 并将时间边界向前移动

```
select mean("water_level") from "h2o_feet" where "location"='coyote_creek' and time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:30:00Z' group by time(18m, 6m)
```

该查询使用InfluxQL函数来计算平均`water_level`，将结果分组为18分钟的时间间隔，并将预设时间边界偏移六分钟。

###### 3. 以12分钟时间间隔分组，并将时间边界向后移动。

```
select mean("water_level") from "h2o_feet" where time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:30:00Z' group by time(12m, -6m)
```

该查询计算`water_level`平均值， 将结果分组为12分钟为时间间隔， 并将预设边界向后移动6分钟。

##### 4. GROUP BY time() 加fill()

`fill()`更改不含数据的时间间隔的返回值。

```
SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(time_interval,[<offset_interval])[,tag_key] [fill(<fill_option>)]
```

默认情况下，没有数据的`GROUP BY time()`间隔返回为null作为输出列中的值。`fill()`更改不含数据的时间间隔返回的值。请注意，如果`GROUP(ing)BY`多个对象（例如，tag和时间间隔），那么`fill()`必须位于`GROUP BY`子句的末尾。

fill的参数：

- 任一数值：用这个数字返回没有数据点的时间间隔
- linear：返回没有数据的时间间隔的[线性插值](https://en.wikipedia.org/wiki/Linear_interpolation)结果。
- none: 不返回在时间间隔里没有点的数据
- previous：返回时间隔间的前一个间隔的数据

###### 1. 例一，参数数字

```
select max("water_level") from "h2o_feet" where "location"='coyote_creek' and time >= '2015-09-18T16:00:00Z' and time <= '2015-09-18T16:42:00Z' group by time(12m) fill(100)
```

使用数值100， 补全没有数据点的时间间隔。

###### 2. 例二，参数linear

```
select mean("tadpoles") from "pond" where time >= '2016-11-11T21:00:00Z' and <= '2016-11-11T22:06:00Z' group by time(12m) fill(linear)
```

线性补全。对没有的数据点时间间隔采用线性补全。

###### 3. 例三， 参数none

````
select max("water_level") from 'h2o_feet' where "location"='coyote_creek' and time >= '2015-09-18T16:00:00Z' and time <= '2015-09-18T16:42:00Z' group by time(12m) fill(none)
````

###### 4. 例四， 参数null

```
select max("water_level") from "h2o_feet" where "location"='coyote_creek' and time >= '2015-09-18T16:00:00Z' and time <= '2015-09-18T16:42:00Z' group by time(12m) fill(null)
```

此参数为将没有数据点的时间间隔显示，不带将不显示。

###### 5. 例五， 参数previous

```
select max("water_level") from "h2o_feet" where "location"='coyote_creek' and time >= '2015-09-18T16:00:00Z' and time <= '2015-09-18T16:42:00Z' group by time(12m) fill(previous)
```

此参数为将没有数据点是时间间隔使用上一个时间间隔数据点填充。

##### 注意点

**`fill()`当没有数据在查询时间范围内时**

目前，如果查询的时间范围内没有任何数据，查询会忽略`fill()`。 这是预期的行为。GitHub上的一个开放[feature request](https://github.com/influxdata/influxdb/issues/6967)建议，即使查询的时间范围不包含数据，`fill()`也会强制返回值。

**`fill(previous)`当前一个结果超出查询时间范围**

当前一个结果超出查询时间范围，`fill(previous)`不会填充这个时间间隔。

**`fill(linear)`当前一个结果超出查询时间范围**

当前一个结果超出查询时间范围，`fill(linear)`不会填充这个时间间隔。



#### INTO 子句

`INTO`子句将查询的结果写入到用户自定义的measurement中。

```
SELECT_clause INTO <measurement_name> FROM_clause [WHERE_clause] [GROUP_BY_clause]
```

`INTO`支持多种格式的measurement。

```
INTO <measurement_name>
```

写入到特定measurement中，用CLI时，写入到用`USE`指定的数据库，保留策略为`DEFAULT`，用HTTP API时，写入到`db`参数指定的数据库，保留策略为`DEFAULT`。

```
INTO <database_name>.<retention_policy_name>.<measurement_name>
```

写入到完整指定的measurement中。

```
INTO <database_name>..<measurement_name>
```

写入到指定数据库保留策略为`DEFAULT`。

```
INTO <database_name>.<retention_policy_name>.:MEASUREMENT FROM /<regular_expression>/
```

将数据写入与`FROM`子句中正则表达式匹配的用户指定数据库和保留策略的所有measurement。 `:MEASUREMENT`是对`FROM`子句中匹配的每个measurement的反向引用。

##### 1. 重命名数据库

```
select * into "copy_NOAA_water_database"."autogen".:MEASUREMENT from "NOAA_water_database"."autogen"./.*/ group by *
```

在InfluxDB中直接重命名数据库是不可能的，因此`INTO`子句的常见用途是将数据从一个数据库移动到另一个数据库。

**请注意**，在运行`INTO`查询之前，`copy_NOAA_water_database`数据库及其`autogen`保留策略y以及copy_NOAA_water_database数据库和autogen保留策略中都必须存在。

反向引用语法（`:MEASUREMENT`）维护目标数据库中的源measurement名称。以下查询不为tag维护series的上下文;tag将作为field存储在目标数据库（`copy_NOAA_water_database`）中：

```
SELECT * INTO "copy_NOAA_water_database"."autogen".:MEASUREMENT FROM "NOAA_water_database"."autogen"./.*/
```

当移动大量数据时，我们建议在`WHERE`子句中顺序运行不同measurement的`INTO`查询并使用时间边界。这样可以防止系统内存不足。下面的代码块提供了这些查询的示例语法：

```
SELECT * 
INTO <destination_database>.<retention_policy_name>.<measurement_name> 
FROM <source_database>.<retention_policy_name>.<measurement_name>
WHERE time > now() - 100w and time < now() - 90w GROUP BY *

SELECT * 
INTO <destination_database>.<retention_policy_name>.<measurement_name> 
FROM <source_database>.<retention_policy_name>.<measurement_name>} 
WHERE time > now() - 90w  and time < now() - 80w GROUP BY *

SELECT * 
INTO <destination_database>.<retention_policy_name>.<measurement_name> 
FROM <source_database>.<retention_policy_name>.<measurement_name>
WHERE time > now() - 80w  and time < now() - 70w GROUP BY *
```

##### 2. 将查询结果写入到一个measurement

```
SELECT "water_level" INTO "h2o_feet_copy_1" FROM "h2o_feet" WHERE "location" = 'coyote_creek'
```

将查询的结果写入`h20_feet_copy_1`。

##### 3. 将查询结果写入一个完全指定的measurement中

```
select "water_level" into "water_else"."autogen"."h2o_feet_copy_2" from "h2o_feet" where "location"='coyote_creek'
```

##### 4. 将聚合结果写入到一个measurement中（采样）

```
SELECT MEAN("water_level") INTO "all_my_averages" FROM "h2o_feet" WHERE "location" = 'coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
```

查询使用InfluxQL函数和`GROUP BY time()`子句聚合数据。它也将其结果写入`all_my_averages`measurement。

该查询是采样的示例：采用更高精度的数据，将这些数据聚合到较低的精度，并将较低精度数据存储在数据库中。 采样是`INTO`子句的常见用例。

##### 5. 将多个measurement的聚合结果写入到不同得到数据库中（逆向引用采样）

```
 SELECT MEAN(*) INTO "where_else"."autogen".:MEASUREMENT FROM /.*/ WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:06:00Z' GROUP BY time(12m)
```

查询使用InfluxQL函数和`GROUP BY time()`子句聚合数据。它会在与`FROM`子句中的正则表达式匹配的每个measurement中聚合数据，并将结果写入`where_else`数据库和`autogen`保留策略中具有相同名称的measurement中。请注意，在运行`INTO`查询之前，`where_else`和`autogen`都必须存在。

该查询是使用反向引用进行下采样的示例。它从多个measurement中获取更高精度的数据，将这些数据聚合到较低的精度，并将较低精度数据存储在数据库中。使用反向引用进行下采样是`INTO`子句的常见用例。

##### 注意点

**丢数据**

如果`INTO`查询在`SELECT`子句中包含tag key，则查询将当前measurement中的tag转换为目标measurement中的字段。这可能会导致InfluxDB覆盖以前由tag value区分的点。请注意，此行为不适用于使用`TOP()`或`BOTTOM()`函数的查询。

要将当前measurement的tag保留在目标measurement中的tag中，`GROUP BY`相关tag key或`INTO`查询中的`GROUP BY *`。

**使用INTO子句自动查询**

本文档中的`INTO`子句部分显示了如何使用`INTO`子句手动实现查询。 有关如何自动执行`INTO`子句查询实时数据，请参阅Continous Queries文档。除了其他用途之外，Continous Queries使采样过程自动化。

#### ORDER BY TIME DESC

默认情况下，InfluxDB以升序的顺序返回结果; 返回的第一个点具有最早的时间戳，返回的最后一个点具有最新的时间戳。 `ORDER BY time DESC`反转该顺序，使得InfluxDB首先返回具有最新时间戳的点。

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] ORDER BY time DESC
```

如果查询包含`GROUP BY`子句,`ORDER by time DESC`必须出现在`GROUP BY`子句之后。如果查询包含一个`WHERE`子句并没有`GROUP BY`子句，`ORDER by time DESC`必须出现在`WHERE`子句之后。

###### 1. 返回最新的点

```
select "water_level" from "h2o_feet" where "location" = 'santa_monica' order by time desc
```

该查询首先从`h2o_feet`measurement返回具有最新时间戳的点。没有`ORDER by time DESC`，查询将首先返回`2015-08-18T00：00：00Z`最后返回`2015-09-18T21：42：00Z`。

###### 2. 返回最新的点并包括 GROUP BY time（）子句

```
selecrt mean("water_level") from "h2o_feet" where time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:42:00Z' group by time(12m) order by time desc
```

该查询在`GROUP BY`子句中使用InfluxQL函数和时间间隔来计算查询时间范围内每十二分钟间隔的平均`water_level`。`ORDER BY time DESC`返回最近12分钟的时间间隔。

#### LIMIT和SLIMIT

`LIMIT <N>`从指定的measurement中返回前`N`个数据点。

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] LIMIT <N>
```

`N`指定从指定measurement返回的点数。如果`N`大于measurement的点总数，InfluxDB返回该measurement中的所有点。请注意，`LIMIT`子句必须以上述语法中列出的顺序显示。

###### 1. 限制返回的点数

```
 SELECT "water_level","location" FROM "h2o_feet" LIMIT 3
```

查询从measurement`h2o_feet`中返回最旧的三个点。

###### 2. 限制返回的点数并包含一个GROUP BY子句

```
SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) LIMIT 2
```

该查询使用InfluxQL函数和GROUP BY子句来计算每个tag以及查询时间内每隔十二分钟的间隔的平均`water_level`。 `LIMIT 2`请求两个最旧的十二分钟平均值。

请注意，没有`LIMIT 2`，查询将返回每个series四个点; 在查询的时间范围内每隔十二分钟的时间间隔一个点。

#### SLIMIT子句

`SLIMIT <N>`返回指定measurement的前个series中的每一个点。

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] GROUP BY *[,time(<time_interval>)] [ORDER_BY_clause] SLIMIT <N>
```

`N`表示从指定measurement返回的序列数。如果`N`大于measurement中的series数，InfluxDB将从该measurement中返回所有series。

有一个[issue](https://github.com/influxdata/influxdb/issues/7571)，要求使用`SLIMIT`来查询`GROUP BY *`。 请注意，`SLIMIT`子句必须按照上述语法中的顺序显示。

##### 1. 限制返回的series的数目

```
SELECT "water_level" FROM "h2o_feet" GROUP BY * SLIMIT 1
```

查询从measurement`h2o_feet`中返回一个series的所有点。

##### 2. 限制返回的series的数据并包含一个GROUP BY time（）子句

```
SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) SLIMIT 1
```

该查询在GROUP BY子句中使用InfluxQL函数和时间间隔来计算查询时间范围内每十二分钟间隔的平均water_level。SLIMIT 1要求返回与measurementh2o_feet相关联的一个series。
请注意，如果没有SLIMIT 1，查询将返回与h2o_feet相关联的两个series的结果：location = coyote_creek和location = santa_monica。

#### LIMIT和SLIMIT组合使用

`SLIMIT <N>`后面跟着`LIMIT <N>`返回指定measurement的个series中的个数据点。

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] GROUP BY *[,time(<time_interval>)] [ORDER_BY_clause] LIMIT <N1> SLIMIT <N2>
```

`N1`指定每次measurement返回的点数。如果`N1`大于measurement的点数，InfluxDB将从该测量中返回所有点。

`N2`指定从指定measurement返回的series数。如果`N2`大于measurement中series联数，InfluxDB将从该measurement中返回所有series。

##### 1. 限制数据点数和series数的返回

```
SELECT "water_level" FROM "h2o_feet" GROUP BY * LIMIT 3 SLIMIT 1
```

查询从measurement`h2o_feet`中的一个series钟返回最老的三个点。

##### 2. 限制数据点数和series数并包含一个GROUP BY time()子句

```
SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) LIMIT 2 SLIMIT 1
```

查询在`GROUP BY`子句中使用InfluxQL函数和时间间隔来计算查询时间范围内每十二分钟间隔的平均`water_level`。`LIMIT 2`请求两个最早的十二分钟平均值，`SLIMIT 1`请求与measurement`h2o_feet`相关联的一个series。

请注意，如果没有`LIMIT 2` `SLIMIT 1`，查询将返回与`h2o_feet`相关联的两个series中的每一个的四个点。

#### OFFSET和SOFFSET子句

`OFFSET`和`SOFFSET`分页和series返回

#### OFFSET子句

`OFFSET <N>`从查询结果中返回分页的N个数据点。

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] LIMIT_clause OFFSET <N> [SLIMIT_clause]
```

`N`指定分页数。`OFFSET`子句需要一个`LIMIT`子句。使用没有`LIMIT`子句的`OFFSET`子句可能会导致不一致的查询结果。

##### 1. 分页数据点

```
SELECT "water_level","location" FROM "h2o_feet" LIMIT 3 OFFSET 3
```

该查询从measurement`h2o_feet`中返回第4，5，6个数据点，如果查询语句中不包括`OFFSET 3`，则会返回measurement中的第1，2，3个数据点。

##### 2. 分页数据点并包含多个子句

```
SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) ORDER BY time DESC LIMIT 2 OFFSET 2 SLIMIT 1
```

次例子包含：

- `SELECT`指明InfluxQL的函数；
- `FROM`指明单个measurement；
- `WHERE`指明查询的时间范围；
- `GROUP BY`将结果对所有tag作group by；
- `GROUP BY time DESC`按照时间戳的降序返回结果；
- `LIMIT 2`限制返回的点数为2；
- `OFFSET 2`查询结果中不包括最开始的两个值；
- `SLIMIT 1`限制返回的series数目为1；

如果没有OFFSET 2， 查询将会返回最先的两个点。

#### SOFFSET子句

`SOFFSET <N>`从查询结果中返回分页的N个series。

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] GROUP BY *[,time(time_interval)] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] SLIMIT_clause SOFFSET <N>
```

`N`指定series的分页数。`SOFFSET`子句需要一个`SLIMIT`子句。使用没有`SLIMIT`子句的`SOFFSET`子句可能会导致不一致的查询结果。

> 注意：如果`SOFFSET`指定的大于series的数目，则InfluxDB返回空值。

##### 1. 分页series

```
SELECT "water_level" FROM "h2o_feet" GROUP BY * SLIMIT 1 SOFFSET 1
```

查询返回与h2o_feet相关的series数据，并返回taglocation = santa_monica。没有SOFFSET 1，查询返回与h2o_feet和location = coyote_creek相关的series的所有数据。

##### 2. 分页series并包含多个子句

```
 SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) ORDER BY time DESC LIMIT 2 OFFSET 2 SLIMIT 1 SOFFSET 1
```

次例子包含：

- `SELECT`指明InfluxQL的函数；
- `FROM`指明单个measurement；
- `WHERE`指明查询的时间范围；
- `GROUP BY`将结果对所有tag作group by；
- `GROUP BY time DESC`按照时间戳的降序返回结果；
- `LIMIT 2`限制返回的点数为2；
- `OFFSET 2`查询结果中不包括最开始的两个值；
- `SLIMIT 1`限制返回的series数目为1；
- `SOFFSET 1`分页返回的series；

如果没有`SOFFSET 2`，查询将会返回不同的series

#### 所有的S 为偏移字段， 不会返回全部字段的信息， 

#### Time Zone 子句

`tz()`子句返回指定时区的UTC偏移量。

```
SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause] tz('<time_zone>')
```

默认情况下，InfluxDB以UTC为单位存储并返回时间戳。 `tz()`子句包含UTC偏移量，或UTC夏令时（DST）偏移量到查询返回的时间戳中。 返回的时间戳必须是RFC3339格式，用于UTC偏移量或UTC DST才能显示。`time_zone`参数遵循[Internet Assigned Numbers Authority时区数据库](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List)中的TZ语法，它需要单引号。

##### 1. 返回UTC偏移到芝加哥时区的数据

```
SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:18:00Z' tz('America/Chicago')
```

查询的结果包括UTC偏移-5个小时的美国芝加哥时区的时间戳。

**Asia/Shanghai**

##### 时间语法

对于大多数`SELECT`语句，默认时间范围为UTC的`1677-09-21 00：12：43.145224194`到`2262-04-11T23：47：16.854775806Z`。 对于具有`GROUP BY time()`子句的`SELECT`语句，默认时间范围在UTC的`1677-09-21 00：12：43.145224194`和now()之间。以下部分详细说明了如何在`SELECT`语句的`WHERE`子句中指定替代时间范围。

##### 绝对时间

用时间字符串或是epoch时间来指定绝对时间

```
SELECT_clause FROM_clause WHERE time <operator> ['<rfc3339_date_time_string>' | '<rfc3339_like_date_time_string>' | <epoch_time>] [AND ['<rfc3339_date_time_string>' | '<rfc3339_like_date_time_string>' | <epoch_time>] [...]]
```

##### 支持的操作符

`=` 等于
`<>` 不等于
`!=` 不等于
`>` 大于
`>=` 大于等于
`<` 小于
`<=` 小于等于

##### 基本算术

所有时间戳格式都支持基本算术。用表示时间精度的字符添加（+）或减去（-）一个时间。请注意，InfluxQL需要+或-和表示时间精度的字符之间用空格隔开。

##### rfc3399时间字符串

```
'YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ'
```

`.nnnnnnnnn`是可选的，如果没有的话，默认是`.00000000`,rfc3399格式的时间字符串要用单引号引起来。

##### 2. 指定一个RFC3339格式的时间间隔

```
SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00.000000000Z' AND time <= '2015-08-18T00:12:00Z'
```

查询会返回时间戳在2015年8月18日00：00：00.000000000和2015年8月18日00:12:00之间的数据。 第一个时间戳（.000000000）中的纳秒是可选的。

请注意，RFC3339日期时间字符串必须用单引号引起来。

##### 3. 对RFC3339格式的时间戳的基本计算

```
SELECT "water_level" FROM "h2o_feet" WHERE time > '2015-09-18T21:24:00Z' + 6m
```

该查询返回数据，其时间戳在2015年9月18日21时24分后六分钟。请注意，`+`和`6m`之间的空格是必需的。

##### epoch_time

Epoch时间是1970年1月1日星期四00:00:00（UTC）以来所经过的时间。默认情况下，InfluxDB假定所有epoch时间戳都是纳秒。也可以在epoch时间戳的末尾包括一个表示时间精度的字符，以表示除纳秒以外的精度。

##### 4. 指定epoch格式的时间间隔

```
SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= 1439856000000000000 AND time <= 1439856720000000000
```

SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= 1439856000000000000 AND time <= 1439856720000000000。

##### 5. 指定epoch格式以秒为精度的时间间隔

```
SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= 1439856000s AND time <= 1439856720s
```

该查询返回的数据的时间戳为2015年8月18日00:00:00和2015年8月18日00:12:00之间。在epoch时间戳结尾处的`s`表示时间戳以秒为单位。

##### 6. 对epoch时间戳的基本计算

```
SELECT "water_level" FROM "h2o_feet" WHERE time > 24043524m - 6m
```

查询返回数据，其时间戳在2015年9月18日21:24:00之前六分钟。请注意，`-`和`6m`之间的空格是必需的。

##### 相对时间

使用`now()`查询时间戳相对于服务器当前时间戳的的数据

```
SELECT_clause FROM_clause WHERE time <operator> now() [[ - | + ] <duration_literal>] [(AND|OR) now() [...]]
```

`now()`是在该服务器上执行查询时服务器的Unix时间。`-`或`+`和时间字符串之间需要空格。

##### 支持的操作符

`=` 等于
`<>` 不等于
`!=` 不等于
`>` 大于
`>=` 大于等于
`<` 小于
`<=` 小于等于

##### 时间字符串

`u`或`µ` 微秒
`ms` 毫秒
`s` 秒
`m` 分钟
`h` 小时
`d` 天
`w` 星期

##### 1. 使用相对时间指定时间间隔

```
select "water_level" from "h2o_feet" where time > now() - 1200d
```

该查询返回过去1200天的数据。

##### 2. 使用绝对和相对时间指定时间间隔

```
SELECT "level description" FROM "h2o_feet" WHERE time > '2015-09-18T21:18:00Z' AND time < now() + 1000d
```

查询返回的数据的时间戳在2015年9月18日的21:18:00到从现在之后1000天之间。

#### 时间使用注意点

**在绝对时间中使用OR**

当前，InfluxDB不支持在绝对时间的`WHERE`子句中使用`OR`。

**再有GROUP BY time() 中查询发生在now（）之后的数据**

大多数`SELECT`语句的默认时间范围为UTC的`1677-09-21 00：12：43.145224194`到`2262-04-11T23：47：16.854775806Z`。对于具有`GROUP BY time()`子句的`SELECT`语句，默认时间范围在UTC的`1677-09-21 00：12：43.145224194`和`now()`之间。

要查询`now()`之后发生的时间戳的数据，具有`GROUP BY time()`子句的`SELECT`语句必须在`WHERE`子句中提供一个时间的上限。

**配置返回的时间戳**

大多数`SELECT`语句的默认时间范围为UTC的`1677-09-21 00：12：43.145224194`到`2262-04-11T23：47：16.854775806Z`。对于具有`GROUP BY time()`子句的`SELECT`语句，默认时间范围在UTC的`1677-09-21 00：12：43.145224194`和`now()`之间。

要查询`now()`之后发生的时间戳的数据，具有`GROUP BY time()`子句的`SELECT`语句必须在`WHERE`子句中提供一个时间的上限。

#### 正则表达式

InluxDB支持在以下场景使用正则表达式：

- 在`SELECT`中的field key和tag key；
- 在`FROM`中的measurement
- 在`WHERE`中的tag value和字符串类型的field value
- 在`GROUP BY`中的tag key

目前，InfluxQL不支持在`WHERE`中使用正则表达式去匹配不是字符串的field value，以及数据库名和retention policy。

> 注意：正则表达式比精确的字符串更加耗费计算资源; 具有正则表达式的查询比那些没有的性能要低一些。

```
SELECT /<regular_expression_field_key> FROM /<regular_expression_measurement>/ WHERE [<tag_key> <operator> /<regular_expression_tag_value>/ | <field_key> <operator> /<regular_expression_field_value>/] GROUP BY /<regular_expression_tag_key>/
```

正则表达式前后使用斜杠`/`，并且使用[Golang的正则表达式语法](http://golang.org/pkg/regexp/syntax/)。

支持的操作符：

`=~` 匹配
`!~` 不匹配

##### 1. 在select 中使用正则表达式指定field key 和 tag key

```
select /l/ from "h2o_feet" limit 1
```

查询选择所有包含`l`的tag key和field key。请注意，`SELECT`子句中的正则表达式必须至少匹配一个field key，以便返回与正则表达式匹配的tag key。

目前，没有语法来区分`SELECT`子句中field key的正则表达式和tag key的正则表达式。不支持语法`/<regular_expression>/::[field | tag]`。

##### 2. 在select中使用正则表达式指定函数里面的field key

```
select distinct(/level/) from "h2o_feet" where "location" = 'santa_monica' and time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:12:00Z'
```

该查询使用InfluxQL函数返回每个包含`level`的field key的去重后的field value。示例数据中没有，所以报错， 如将distinct和括号去掉，即可匹配到level。

##### 3. 在from 中使用正则比配measurement

```
select mean("degrees") from /temperature/
```

该查询使用InfluxQL函数计算在数据库`NOAA_water_database`中包含`temperature`的每个measurement的平均`degrees`。

##### 4. 在WHERE中使用正则指定无值的tag

```
select * from "h2o_feet" where "location" !~ /./
```

该查询从measurementh2o_feet中选择所有数据，其中tag location没有值。NOAA_water_database中的每个数据点都具有location这个tag。

##### 5. 在where中使用正则指定有值的tag

```
select mean("water_level") from "h2o_feet" where "location" =~ /./
```

查询使用InfluxQL函数计算所有`location`这个tag的数据点的平均`water_level`。

##### 6. 在where中使用正则指定field value

```
select mean("water_level") from "h2o_feet" where "location" = 'santa_monica' and "level description" =~ /between/
```

该查询使用InfluxQL函数计算所有字段`level description`的值含有`between`的数据点的平均`water_level`。

##### 7. 在group by中使用正则指定tag key

````
select first("index") from "h2o_quality" group by /1/
````

该查询使用InfluxQL函数查询每个tag key包含字母`l`的tag的第一个`index`值。

#### 数据类型和转换

在`SELECT`中支持指定field的类型，以及使用`::`完成基本的类型转换。

field的value支持浮点，整数，字符串和布尔型。`::`语法允许用户在查询中指定field的类型。

> 注意：一般来说，没有必要在SELECT子句中指定字段值类型。 在大多数情况下，InfluxDB拒绝尝试将字段值写入以前接受的不同类型的字段值的字段的任何数据。字段值类型可能在分片组之间不同。在这些情况下，可能需要在SELECT子句中指定字段值类型。

```
SELECT_clause <field_key>::<type> FROM_clause
```

`type`可以是`float`，`integer`，`string`和`boolean`。在大多数情况下，如果`field_key`没有存储指定`type`的数据，那么InfluxDB将不会返回数据。

##### 1. 例一

```
select "water_level"::float from "h2o_feet" limit 4
```

查询返回field key`water_level`为浮点型的数据。

##### 类型转换

`::`语法允许用户在查询中做基本的数据类型转换。目前，InfluxDB支持冲整数转到浮点，或者从浮点转到整数。

```
SELECT_clause <field_key>::<type> FROM_clause
```

`type`可以是`float`或者`integer`。

如果查询试图把整数或者浮点数转换成字符串或者布尔型，InfluxDB将不会返回数据。

##### 1. 例一

```
select "water_level"::integer from "h2o_feet" limit 4
```

将浮点数转换成整形。

##### 2. 例二（目前不支持）

```
select "water_level"::string from "h2o_feet" limit 4
```

浮点数转化成字符串。所有的返回为空。



#### 多语句

用分号`;`分割多个`SELECT`语句。

##### 1. 例子

```
select mean("water_level") from "h2o_feet"; select "water_level" from "h2o_feet" limit 2
```



#### 子查询

子查询是嵌套在另一个查询的`FROM`子句中的查询。使用子查询将查询作为条件应用于其他查询。子查询提供与嵌套函数和SQL`HAVING`子句类似的功能。

```
SELECT_clause FROM ( SELECT_statement ) [...]
```

InfluxDB首先执行子查询，再次执行主查询。

主查询围绕子查询，至少需要`SELECT`和`FROM`子句。主查询支持本文档中列出的所有子句。

子查询显示在主查询的`FROM`子句中，它需要附加的括号。 子查询支持本文档中列出的所有子句。

InfluxQL每个主要查询支持多个嵌套子查询。 多个子查询的示例语法：

```
SELECT_clause FROM ( SELECT_clause FROM ( SELECT_statement ) [...] ) [...]
```

##### 1. 计算多个max值的sum（）

```
select sum("max") from (select max("water_level") from "h2o_feet" group by "location")
```

该查询返回`location`的每个tag值之间的最大`water_level`的总和。

##### 2. 计算两个field的差值的mean（）

```
select mean("difference") from (select "cats" - "dogs" as "difference" from "pet_daycare")
```

查询返回measurement`pet_daycare``cats`和`dogs`数量之间的差异的平均值。

##### 3. 计算mean（） 然后将这些平均值作为条件

```
select "all_the_means" from (select mean("water_level") as "all_the_means" from "h2o_feet" where time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' group by time(12m)) where "all_the_means" > 5
```

该查询返回`water_level`的平均值大于5的所有平均值。

InfluxDB首先执行子查询。子查询从2015-08-18T00：00：00Z到2015-08-18T00：30：00Z计算`water_level`的`MEAN()`值，并将结果分组为12分钟。

接下来，InfluxDB执行主查询，只返回大于5的平均值。请注意，主查询将`all_the_means`指定为`SELECT`子句中的字段键。

##### 4. 计算多个DERIVATIVE（）值得sum（）

```
select sum("water_level_derivative") as "sum_derivative" from (select derivative(mean("water_level")) as "water_level_derivative" from "h2o_feet" where time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' group by time(12m), "location") group by "location"
```

查询返回每个tag `location`的平均`water_level`的导数之和。InfluxDB首先执行子查询。子查询计算以12分钟间隔获取的平均`water_level`的导数。它对`location`的每个tag value进行计算，并将输出列命名为`water_level_derivative`。

接下来，InfluxDB执行主查询，并计算`location`的`water_level_derivative`值的总和。请注意，主要查询指定了`water_level_derivative`，而不是`water_level`或者`derivative`，作为`SUM()`函数中的字段键。

#### 子查询注意点

InfluxQL支持在每个主查询中嵌套多个子查询：

```
select_clause from (select_clause from (select_statement)[...])[...]
					-----------------    ----------------
					 Subquery 1				Subquery 2
```

InfluxQL**不支持**每个子查询中多个`SELECT`语句：

```
select_clause from (select_statement; select_statement)[...]
```

如果出现， 系统将会返回一个解析错误。



### schema 查询（查看数据库信息）

#### show databases

返回当前示例上的所有的数据库

```
show databases
```

#### show retention policies

返回指顶数据库的保留政策的列表

```
show retention policies on [database name]
# OR
use [database name]
show retention policies
```

#### show series

返回指定数据库的series列表

```
SHOW SERIES [ON <database_name>] [FROM_clause] [WHERE <tag_key> <operator> [ '<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`FROM`，`WHERE`，`LIMIT`和`OFFSET`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW SERIES`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于
`<>` 不等于
`!=` 不等于
`=~` 匹配
`!~` 不匹配

##### 1. 运行带有ON 子句的showseries

```
show series on NOAA_water_database
```

查询的输出类似于行协议格式。第一个逗号之前的所有内容都是measurement名称。第一个逗号后的所有内容都是tag key或tag value。 `NOAA_water_database`有五个不同的measurement和14个不同的series。

##### 2.  运行不带ON

```
use NOAA_water_database
show series
```

##### 3. 运行多个子句

```
show series on NOAA_water_database from "h2o_quality" where "location" = 'coyote_creek' limit 2
```

查询返回数据库`NOAA_water_database`中与measurement `h2o_quality`相关联的并且tag为`location = coyote_creek`的两个series。

#### shwo measurements

返回指定数据库的measurement列表

```
SHOW MEASUREMENTS [ON <database_name>] [WITH MEASUREMENT <regular_expression>] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`WITH`，`WHERE`，`LIMIT`和`OFFSET`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW MEASUREMENTS`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于
`<>` 不等于
`!=` 不等于
`=~` 匹配
`!~` 不匹配

##### 1. 运行带ON

```
show measurements
```

##### 2. 运行不带ON

```
use NOAA_water_database
show measurement
```

##### 3. 运行多个子句

```
show measurements on NOAA_water_database with measurement =~/h2o.*/ limit 2
```

#### show tag keys

返回数据库的tag key列表。

```
SHOW TAG KEYS [ON <database_name>] [FROM_clause] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`FROM`和`WHERE`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW TAG KEYS`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于
`<>` 不等于
`!=` 不等于
`=~` 匹配
`!~` 不匹配

##### 1. 运行带on

```
show tag keys on "NOAA_water_database"
```

##### 2. 运行不带on

```
use NOAA_water_database
show tag keys
```

##### 3. 运行多个子句

```
show tag keys on "NOAA_water_database" from "h2o_quality" limit 1 offset 1
```

#### show tag values

返回数据库中指定的tag key的tag value列表

```
SHOW TAG VALUES [ON <database_name>][FROM_clause] WITH KEY [ [<operator> "<tag_key>" | <regular_expression>] | [IN ("<tag_key1>","<tag_key2")]] [WHERE <tag_key> <operator> ['<tag_value>' | <regular_expression>]] [LIMIT_clause] [OFFSET_clause]
```

`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`WITH`子句是必须的，它支持指定一个单独的tag key、一个表达式或是多个tag key。

`FROM`、`WHERE`、`LIMIT`和`OFFSET`子句是可选的。 `WHERE`子句支持tag比较; field比较对`SHOW TAG KEYS`查询无效。

`WHERE`子句中支持的运算符：

`=` 等于
`<>` 不等于
`!=` 不等于
`=~` 匹配
`!~` 不匹配

##### 1.运行带on

```
show tag values on "NOAA_water_database" with key = 'randtag'
```

##### 2. 运行不带ON

```
use NOAA_water_database
show tag values with key= "randtag"
```

##### 3. 运行多个子句

```
show tag values on "NOAA_water_database" with key in ("location", "randtag") where "randtag"=~ /./ limit 2
```

#### show field keys

返回field key以及其field value 的数据类型

```
SHOW FIELD KEYS [ON <database_name>] [FROM <measurement_name>]
```

`ON <database_name>`是可选的。如果查询不包括`ON <database_name>`，则必须在CLI中使用`USE <database_name>`指定数据库，或者在HTTP API请求中指定`db`查询字符串参数。

`FROM`子句也是可选的。

##### 1. 运行带ON

```
show field keys on "NOAA_water_database"
```

##### 2. 运行不带on

```
use [database name]
show fields keys
```

#### 3. 运行带有from的

```
show field keys ON "NOAA_water_database" from "h2o_feet"
```

返回数据库`NOAA_water_database`中measurement为`h2o_feet`的对应的field key以及其数据类型。

#### 注意点

**show field keys 和field类型的差异**。

field value的数据类型在同一个shard里面一样但是在多个shard里面可以不同，`SHOW FIELD KEYS`遍历每个shard返回与field key相关的每种数据类型。