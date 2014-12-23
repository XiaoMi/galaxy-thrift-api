<!--
pandoc README.md --highlight-style tango -o README.html
pandoc README.md --highlight-style tango -o README.pdf -s --latex-engine=xelatex --template=template.tex
 -->
小米结构化数据存储开发文档
============

如果您在集成过程中遇到任何问题，都可以添加QQ群：385428920 ，群中会有
工程师解答您的问题。

[SDK地址](https://github.com/XiaoMi)

1. 功能介绍
--------------
结构化数据存储(SDS)是一种高效，全面托管的分布式NoSQL数据库服务，为应
用开发者提供快速、安全的数据存储服务。主要功能有：

  * **弹性可扩展** - 表空间和吞吐量按需配置，自动扩展，无需人工干预
  * **高可用** - 安全高效，自动故障迁移与负载均衡，数据强一致保证
  * **简单的数据模型** - 提供类似关系数据库的表格存储模型，支持完善的数据类
  型支持，降低了开发者的门槛。在支持分布式存储的基础上，保留了关系数据
  库的强一致的局部二级索引以及类SQL Where语句的组合条件查询，保证开发
  效率
  * **离线分析支持** - 支持主备集群复制，Java SDK内置MapReduce/Hive支持，为
  大规模数据处理提供了便利
  * **完善的认证授权机制** - 支持多种身份认证机制，用户授权，保障用户数据的安
  全隔离
  * **多语言SDK支持** - 支持多种语言SDK
  
2. 数据模型
------------
结构化存储的数据模型是表格模型，属于强schema模型，即一个表由多行相同格式的记录组成。
从逻辑上，每行数据是一组属值的集合，属性支持如下数据类型：

数据类型 | Java SDK  | PHP SDK | Python SDK |  Go SDK  | 备注
-------- | --------- | ------- | ---------- | -------- | ---
BOOL     | boolean   | boolean | bool       |  bool    | -
INT8     | byte      | integer | int        |  int8    | -
INT16    | short     | integer | int        |  int16   | -
INT32    | int       | integer | int        |  int32   | -
INT64    | long      | integer | long       |  int64   | -
FLOAT    | float     | 不支持  | float      |  float32 | -
DOUBLE   | double    | double  | float      |  float64 | -
STRING   | String    | string  | str        |  string  | 不能包含'\\0'
BINARY   | byte[]    | string  | string     |  []byte  | -
RAWBINARY| byte[]    | 不支持  | 不支持     |  []byte  | 不能作为实体组键，主键和二级索引属性
集合类型 | List      | array   | list       |  slice   | 支持二级索引

表的数据在物理组织上支持两种索引结构：
  * **主键 (Primary Key)** - 必选，由一到多个表属性组成
    * 所有其余属性按照主键顺序存储，根据主键读取/写入数据时，记1个单位的读/写配额
  * **局部二级索引 (Local Secondary Index)** - 可选，使用局部二级索引必须要定义实体组键，由一个到多个表属性组成。索引分为*Lazy*，*Eager*和*Immutable*三种类别
    * *Lazy索引* - 此类型的索引不支持投影属性和唯一索引。写入时，消耗1个写配额。读取时，消耗2个读配额
    * *Eager索引* - 此类型的索引可以支持唯一索引(*Unique Index*)，并且可以定义一组属性作为
    投影(*Projection*)，与索引存储在一起（可视为主记录中对应属性的一份拷贝，并且属性值保持强
    一致）。索引写入时，会消耗1个单位的读配额和1个单位的写配额，如果是唯一索引，则需要消耗1个
    额外的读配额。通过索引读取时，如果只需要读取索引中的属性或投影属性，则只消耗1个读配额。否则，
    如果请求读取的属性需要从主记录中读取，则总共需要消耗2个读配额
    * *Immutable索引* - 此类型的索引适合只读数据，即写入后数据不会再修改(**需要开发者自己来保证**)。
    此类型索引支持投影，但不支持唯一索引。写入时，消耗1个写配额。读取时，消耗1个读配额(但需要
    读取非投影的属性的时候则为2个)

另外，在表的定义中，可以将一到多个表属性定义为 **实体组键(Entity Group Key)**，在物理组织上，
实体组键是记录主键和二级索引的前缀。对于实体组键，可以定义是否进行哈希分布，开启此选项后，
会添加取值范围为256的哈希值，从而实现负载均衡的效果，可用于消除请求热点，建议开启。需要注意，
打开此选项后，表中的数据将不再保证全局有序。一般情况下，没有特殊原因，强烈建议设置实体组键并开启哈希
(**设计表的Schema以及估算请求量时，应保证每个实体组上的单位时间消耗的读写配额峰值尽可能的小，原则上
不应不超过1000/秒，否则将不保整个表的读写请求配额。实体组的数量的多少不影响性能，所以在满足业务查询
功能前提下，应该尽可能降低每个实体组上的读写配额，避免读写热点，尤其避免同时大量读写一个或少数几个
实体组中的记录**)。
当实体组键开启哈希分布时，主键需要包含至少一个属性，如果schema中没有可以作为主键的属性，建议在主键中
使用一个默认的占位属性(例如，如果一个表只有一个userId作为实体组键属性，那么可以添加一个额外的recordId
属性作为主键，取值为常数0，表示某个用户下的第1条记录)。

对于实体组键，主键以及二级索引的属性的编码(*KeySpec*)，可以通过*asc*选择决定是否是升序还是降序(例如，
整型降序编码时，其排序为..., 2, 1, 0, -1, ...与正序相反)。对于扫描(Scan)操作，可以通过通过设置
*reverse*选项进行正序和逆序扫描，但是正序扫描效率高于逆序扫描，所以定义表结构的时候，需要根据关键路径
上的查询模式来决定编码顺序，尽量保证关键路径是正序扫描。

主记录（主键）行的组织形式为：

```
    [实体组前缀] [主键属性1] ... [主键属性m]:
    {表属性1, ..., 表属性p}
```

其中实体组前缀是可选的，仅当表定义了实体组键时存在。二级索引行的组织形式为：

```
    [实体组前缀] [索引id] [二级索引属性1] ... [二级索引属性n] [主键属性1] ... [主键属性m]:
    {投影属性1, ..., 投影属性q}
```

前面已经提到，投影属性为主记录行的副本，与主数据行总是保证一致，且仅当eager索引才可以定义投影属性。

其中实体组键前缀的组成形式为：

```
    [实体组哈希值] [实体组键属性1] ... [实体组键属性k]
```

开发者可以根据数据的读取模式，选择是否支持实体组键，以及是否定义二级索引。同时，二级索引需要根据
读写的比例选择类型，例如 *Lazy索引* 适合写入较多，读取较少且对延迟不敏感的情况。*Eager索引* 与
投影配合使用，适合读取相对频繁或者对读延迟比较敏感的场合。而只读数据则适合采用*Immutable索引*。
需要注意的是，**目前不支持动态修改表的实体组键(包括哈希分布选项)，主键，除索引类型外的其
他二级索引选项)，以及属性的数据类型和编码方式**，建表时需要慎重选择。表的普通属性可以增删，
但数据访问会有短暂停服（平均在2-3秒），建议在业务低峰期操作。
二级索引的类型可以修改，但仅允许*Immutable*改为*Eager*，适合当表中数据由不可更新改为可写的情况。
一种常见的场景是，建表初期导入已有数据，此时数据是只读的，不会存在更新的情况(**需要开发者自己保证**)，
此时索引定义为*Immutable*，可以节约读配额，数据导入完成后开始线上服务之前，将索引改为*Eager*模式。

3. 一致性模型
---------------
SDS存储是强一致的模型，即后续发生的读总能读到以前写入的数据。数据存储三备份，并且会异步的同步到备集群，备集群目前只提供只读功能，用于离线分析。

4. 事务性
-----------
###4.1 行级别的事务保证###
对于同一次*put*（非*batch*）的记录数
据，能够保证原子性，例如，两个并发的*put*分别写入同一行的两个属性*p*和*q*: (p1, q1)和(p2, q2)，
最终结果不会出现(p1, q2)或者(p2, q1)的情况。

###4.2 实体组内的事务保证###
属于同一事务组内的记录行，可以支持*batch*的原子性，可以在建表的时候进行配置是否开启，如果需要定义二级索引，则必须开启*batch*原子性。同一事务组内可以支持局部二级索引，并且索引跟主记录之间是强一致的。

###4.3 自增操作支持###
目前支持在整形数据类型上的自增操作

###4.4 条件修改###
SDS支持条件修改(*put*和*delete*)，这样可以在应用层实现自己的同步逻辑(比如锁)。

5. 示例
------------
以记事本应用为例，此应用支持通过浏览器和移动客户端两种方式存取数据。下面是基于关系数据库的表设计：

```sql
CREATE TABLE note (
  userId VARCHAR(64) NOT NULL,
  noteId INT8 NOT NULL, -- 自增ID
  title VARCHAR(256),
  content VARCHAR(2048),
  version INT,
  mtime BIGINT, -- 自增唯一的ID
  category VARCHAR(16),
  
  PRIMARY KEY(userId, noteId),
  INDEX(userId, mtime), -- 修改时间索引
  INDEX(userId, category) -- 类别标签索引
);
```

假设应用有四种访问模式：

  * 根据*noteId*查询一条笔记的详细信息
  * 根据*noteId*保存一条笔记的详细信息(可对*version*进行条件写入，以检测是否有并发修改)
  * 根据笔记的修改时间进行扫描分页查询，仅显示title，用户可点击进一步查看一条笔记详细信息
  * 根据笔记的类别标签进行查询

###5.1 表定义###
对于此应用，可以转化为SDS的表定义：

  * *实体组键* - *userId*，开启哈希分布
  * *主键* - *noteId*，逆序编码
  * *修改时间索引* - *mtime*，逆序编码，*eager*类型的索引。并对*title*和*noteId*做投影
  * *类别索引* - *category*，*lazy*类型的索引

###5.2 创建client###
以PHP为例，可通过以下代码创建client:
```php
$credential = new Credential(
  array(
    "type" => UserType::APP_SECRET,
    "secretKeyId" => "替换为自己帐号的任一AppKey",
    "secretKey" => "替换为对应的AppSecret"
  )
);
$clientFactory = new ClientFactory($credential, false, true); // verbose off
$endpoint = "https://sds.api.xiaomi.com";
$adminClient = $clientFactory->newAdminClient($endpoint .
  $GLOBALS['Common_CONSTANTS']['ADMIN_SERVICE_PATH'],
  $GLOBALS['Common_CONSTANTS']['DEFAULT_ADMIN_CLIENT_TIMEOUT'],
  $GLOBALS['Common_CONSTANTS']['DEFAULT_CLIENT_CONN_TIMEOUT']);
$tableClient = $clientFactory->newTableClient($endpoint .
  $GLOBALS['Common_CONSTANTS']['TABLE_SERVICE_PATH'],
  $GLOBALS['Common_CONSTANTS']['DEFAULT_CLIENT_TIMEOUT'],
  $GLOBALS['Common_CONSTANTS']['DEFAULT_CLIENT_CONN_TIMEOUT']);
```

可以根据自己的需求，配置各个参数，例如，可以通过*ClientFactory*的第二个参数配置是否对操作超时
自动进行重试(如果自动重试需要注意操作的幂等性)，另外还可以根据安全策略和性能考量选择http或者
https方式访问。此外，还可以自定义底层socket的建立连接和读写的超时时间。对于Java SDK来说，还可以
配置HttpClient是够为线程安全的，以及连接池的参数。

###5.3 建表###
见表过程可以通过开发者站的页面操作完成，如果需要，可以通过代码方式创建：
```php
$tableName = "php-note";
// create table
$tableSpec = new TableSpec(array(
  'schema' => new TableSchema(array(
      'entityGroup' => new EntityGroupSpec(array(
          'attributes' => array(new KeySpec(array('attribute' => 'userId'))),
          'enableHash' => true, // hash distribution
        )),
      // Creation time order
      'primaryIndex' => array(
        new KeySpec(array('attribute' => 'noteId', 'asc' => false)),
      ),
      'secondaryIndexes' => array(
        // Default display order, sorted by last modify time
        'mtime' => new LocalSecondaryIndexSpec(array(
            'indexSchema' => array(
              new KeySpec(array('attribute' => 'mtime', 'asc' => false)),
            ),
            'projections' => array('title', 'noteId'),
            'consistencyMode' => SecondaryIndexConsistencyMode::EAGER,
          )),
        // Search by category
        'cat' => new LocalSecondaryIndexSpec(array(
            'indexSchema' => array(
              new KeySpec(array('attribute' => 'category')),
            ),
            'consistencyMode' => SecondaryIndexConsistencyMode::LAZY,
          )),
      ),
      'attributes' => array(
        'userId' => DataType::STRING,
        'noteId' => DataType::INT64,
        'title' => DataType::STRING,
        'content' => DataType::STRING,
        'mtime' => DataType::INT64,
        'category' => DataType::STRING,
      ),
    )),
  'metadata' => new TableMetadata(array(
      'quota' => new TableQuota(array('size' => 100 * 1024 * 1024)),
      'throughput' => new ProvisionThroughput(array(
          'readQps' => 100,
          'writeQps' => 200
        ))
    ))
));
```

###5.4 写数据###
*Put*数据时，需要指定需要指定全部的实体组键(如果有定义)和主键的属性值，以及部分属性。特别的，对于*lazy*索引，所有写入的属性集合必须包含其全部或0个索引属性，即不允许只写入其部分属性。

对于笔记应用，插入和修改操作如下，其中修改操作利用了*条件Put*来检测并发修改冲突，代码如下：
```php
$categories = array("work", "travel", "food");
for ($i = 0; $i < 20; $i++) {
  $version = 0; // initial version
  $insert = new PutRequest(array(
    "tableName" => $tableName,
    "record" => Array(
      "userId" => DatumUtil::datum("user1"),
      "noteId" => DatumUtil::datum($i),
      "title" => DatumUtil::datum("Title $i"),
      "content" => DatumUtil::datum("note $i"),
      "version" => DatumUtil::datum($version),
      "mtime" => DatumUtil::datum($i * $i % 10),
      "category" => DatumUtil::datum($categories[rand(0, sizeof($categories) - 1)]),
    )));
  // insert
  $tableClient->put($insert);

  $put = $insert;
  $put->record["version"] = DatumUtil::datum($version + 1);
  $put->record["content"] = DatumUtil::datum("new content $i");
  $put->record["mtime"] = DatumUtil::datum($i * $i % 10 + 1);
  $put->condition = new SimpleCondition(array(
    "operator" => OperatorType::EQUAL,
    "field" => "version",
    "value" => DatumUtil::datum($version)
  ));
  // update if note is not concurrently modified
  $tableClient->put($put);
  echo "update note without conflict? " . $result->success . "\n";
}
```

###5.5 根据主键读取###
根据主键读取数据，需要指定全部的实体组键(如果有定义)和主键的属性值。其中对于返回值，可以通过*attributes*属性指定返回部分属性，没有指定时，则表示获取所有的属性：
```php
$get = new GetRequest(array(
  "tableName" => $tableName,
  "keys" => array(
    "userId" => DatumUtil::datum("user1"),
    "noteId" => DatumUtil::datum(rand(0, 10)),
  ),
));

$result = $tableClient->get($get);
```

###5.6 扫描操作###
除随机读一条数据之外，还可以进行范围扫描查询。可以选择使用主键(*indexName*不指定表示通过主键查询)
和二级索引查询。扫描可以指定查询范围，不指定时表示全表扫描。
查询范围定义的区间是*[startKey, stopKey)*，即两者全部属性都指定且相同时
查询的范围为空。其中*startKey*和*stopKey*可以只指定主键或者对应索引的部分属性(必须为前缀)，指定
前缀时，查询范围的区间为
*[startKey前缀 + 后缀可能的最小值, stopKey前缀 + 后缀可能的最大值 + 1)*。例如，
*startKey = {userId = "user1"}, stopKey = {userId = "user1"}* 表示查询*user1*下的所有笔记。

```php
$scan = new ScanRequest(array(
  "tableName" => $tableName,
  "indexName" => "mtime",
  "startKey" => array(
    "userId" => DatumUtil::datum("user1"),
  ),
  "stopKey" => array(
    "userId" => DatumUtil::datum("user1"),
  ),
  "condition" => "title REGEXP '.*[0-5]' AND noteId > 5",
  "attributes" => array("noteId", "title", "mtime"),
  "limit" => 50 // max records returned for each call, when used with TableScanner
  // this will serve as batch size
));

$scanner = new TableScanner($tableClient, $scan);

foreach ($scanner->iterator() as $k => $v) {
  echo "$k: " . DatumUtil::value($v['noteId'])
    . " [" . DatumUtil::value($v['title']) . "] "
    . DatumUtil::value($v['mtime']) . "\n";
}
```

另外，扫描操作还可以指定过滤条件，需要注意的是，被条件过滤掉的记录也要计算读配额(根据索引类型
记1个或2个读配额)。过滤条件的语法类似SQL的Where语句。需要注意，属性不存在表示值为*null*，除*isnull*和*notnull*操作符之外的所有其他表达式和函数调用结果都为*null*，如果表达式最终的值为*null*，则表达式的计算结果为*false*。

例如对于表中的一条记录记录：

```c
{
  i : 20,       // int32
  true : true,  // boolean
  false : false // boolean
}
```

判定结果取值为true的条件：

```sql
    true 
    [true] 
    1 == 1 
    -1 >= -1 
    1 <= 1 
    1 != 2 
    1 < 2 
    2 > 1 
    2.0 > 1 
    1.0 < 2e5 
    1 + 2 * 10 / 2 - 4 % 3 == 10 
    (1 + 2) * ((10 / 2) - 4) % 3 == 0 
    not(false) 
    i + 10 > [i] + 9 
    i notnull 
    unknown isnull 
    'hello world' == 'hello ' || 'world' 
    '''''' == '''' || '''' 
    'hello world' regexp 'hello [a-z]+' 
    true and not false or unknown 
    1 < 2 and not 2 < 1 or 2 / 0 
    string(1) == '1' 
    lower('AbC') == 'abc' 
    upper('aBc') == 'ABC' 
    length('') == 0 
    length('''') == 1 
    length('abc') == 3 
    substr('abc', 1, 2) == 'b' 
    substr('abc', 5, 6) == '' 
    trim(' a ') == 'a' 
    max(-1, 2) == 2 
    min(-1, 2) == -1 
    abs(-1) == 1 
    abs(pow(2, 3) - 8) < 1e-5 
    abs(log(2 * 3) - log(2) - log(3)) < 1e-5 
    rand() != rand() 
    rand(1) >= 0 and rand(1) < 1
```

判定结果取值为false(包含最终表达式取值为null的情况)的条件：

```sql
    false
    [false]
    1 != 1
    1 > 1
    1 < 1
    1 == 2
    1 > 2
    2 < 1
    2.0 < 1
    'hello world' regexp 'bonjour.*'
    i isnull
    unknown notnull
    NOT true
    unknown
    true and unknown
```

如果定义了哈希分布，跨实体组进行扫描需要先分别确定在每个哈希桶中的范围分布，再分别对每个哈希桶进行局部扫描，最后对所有桶内扫描结果进行归并排序，因此会
引入较大性能开销，建议谨慎使用。
注：这种场景建议用户采用结构化存储MapReduce API(即将上线)，MapReduce分布式计算框架将上述步骤分布到多台机器并行执行，从而极大提高扫描性能。

6. API文档与SDK支持与开发
----------------
详细的API文档在[galaxy-thrift-api](https://github.com/XiaoMi/galaxy-thrift-api)子项目中。
目前提供了[Java](https://github.com/XiaoMi/galaxy-sdk-java)(包括Android)，
[PHP](https://github.com/XiaoMi/galaxy-sdk-php)，
[Python](https://github.com/XiaoMi/galaxy-sdk-python)，
[Go](https://github.com/XiaoMi/galaxy-sdk-go)以及
[Javascript](https://github.com/XiaoMi/galaxy-sdk-javascript)语言的SDK，其他语言的SDK如有需求可以自己实现。
结构化存储的API是通过Thrift定义，传输协议采用`TJSONProtocol`。

现在以`describeTable` API为例，介绍SDK实现：

```php
// 测试时请替换真实的AppKey/AppSecret
$appKey = "12345678901234567890";
$appSecret = "WOqDHS8AbBHGhaOD2pvmCQ==";

$credential = new Credential(
  array(
    "type" => UserType::APP_SECRET,
    "secretKeyId" => $appKey,
    "secretKey" => $appSecret
  )
);
$clientFactory = new ClientFactory($credential, true, true);
$endpoint = "https://sds.api.xiaomi.com";
$adminClient = $clientFactory->newAdminClient($endpoint .
  Constant::get('ADMIN_SERVICE_PATH'), 5, 5);

$adminClient->describeTable("test");
```

`describeTable`调用对应的HTTP请求：
```yaml
POST /v1/api/admin HTTP/1.1
Host: sds.api.xiaomi.com
Accept: application/x-thrift
User-Agent: PHP-SDK/1.0.d0bc1b06 PHP/5.3.10-1ubuntu3.15
Content-Type: application/x-thrift
Content-Length: 44
X-Xiaomi-Timestamp: 1416822141
X-Xiaomi-Content-MD5: 43a0cd31f7648b16a45f0371d58f8689
Authorization: {"1":{"str":"SDS-V1"},"2":{"i32":10},"3":{"str":"12345678901234567890"},"5":{"str":"92543e103926a42afd4c2644f7d793753239333f"},"6":{"i32":2},"7":{"lst":["str",3,"Host","X-Xiaomi-Timestamp","X-Xiaomi-Content-MD5"]}}

[1,"describeTable",1,0,{"1":{"str":"test"}}]
```

对应的服务端响应：
```yaml
HTTP/1.1 200 OK
Server: Tengine/2.0.1
Date: Mon, 24 Nov 2014 09:42:13 GMT
Content-Type: application/x-thrift
Transfer-Encoding: chunked
Connection: keep-alive
Keep-Alive: timeout=360
X-Xiaomi-Error-Code: 0
X-Xiaomi-Timestamp: 1416822133
Cache-Control: no-cache, no-store, must-revalidate
Pragma: no-cache
Expires: 0

[1,"describeTable",2,0,{"1":{"rec":{"1":{"i32":26},"2":{"str":"The table which you are attempting to access does not exist"},"3":{"str":"Table not found [test]"},"4":{"str":"omqt46b7"}}}}]
```

###6.1 身份认证###
身份认信息是通过HTTP头`Authorization`来传递。`Authorization`头由`HttpAuthorizationHeader`对象通过`TJSONProtocol`序列化得到：

```thrift
/**
 * Authorization头包含的内容
 */
struct HttpAuthorizationHeader {
  1: required string version = "SDS-V1",
  2: required UserType userType = UserType.APP_ANONYMOUS,
  3: required string secretKeyId,
  /**
   * 直接使用sercetKey，此项被设置时，signature将被忽略，
   * 非安全传输应使用签名
   */
  4: optional string secretKey,
  /**
   * 如secretKey未设置，则认为使用签名，此时必须设置，
   * 被签名的正文格式：header1[\nheader2[\nheader3[...]]]，
   * 如使用默认SUGGESTED_SIGNATURE_HEADERS时为：$host\n$timestamp\n$md5
   */
  5: optional string signature,
  /**
   * 签名HMAC算法，客户端可定制，
   * 使用签名时必须设置
   */
  6: optional MacAlgorithm algorithm,
  /**
   * 包含所有签名涉及到的部分，建议使用SUGGESTED_SIGNATURE_HEADERS，
   * 服务端未强制必须使用所列headers，定制的client自己负责签名的安全强度，
   * 使用签名时必须设置
   */
  7: optional list<string> signedHeaders = [],
}
```

其中`userType`，`secretKeyId`，`secretKey`为对应的认证信息。对于`APP_SECRET`方式登录，`secretKeyId`，`secretKey`分别为`AppKey`和`AppSecret`。
对于非安全传输通道(HTTP)，不应当直接传递`secretKey`，而应采用通过`secretKey`对请求进行签名
([HMAC: Hash-based message authentication code](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code))的方式进行身份认证。


例如，对于示例中的请求：

```
[1,"describeTable",1,0,{"1":{"str":"test"}}]
```

其MD5值为`43a0cd31f7648b16a45f0371d58f8689`。计算签名需要的HTTP头信息如下：

```yaml
Host: sds.api.xiaomi.com
X-Xiaomi-Timestamp: 1416822141
X-Xiaomi-Content-MD5: 43a0cd31f7648b16a45f0371d58f8689
```

其签名`HMAC_SHA1("WOqDHS8AbBHGhaOD2pvmCQ==", "sds.api.xiaomi.com\n1416822141\n43a0cd31f7648b16a45f0371d58f8689")`为`92543e103926a42afd4c2644f7d793753239333f`。
最终得到`HttpAuthorizationHeader`结构：

```c
{
  version: "SDS-V1",
  userType: UserType.APP_SECRET,
  secretKeyId: "12345678901234567890",
  signature: "92543e103926a42afd4c2644f7d793753239333f",
  algorithm: MacAlgorithm.HmacSHA1,
  signedHeaders: ["Host", "X-Xiaomi-Timestamp", "X-Xiaomi-Content-MD5"]
}
```
最终通过`TJSONProtocol`序列化得到`Authorization`头：

```yaml
Authorization: {"1":{"str":"SDS-V1"},"2":{"i32":10},"3":{"str":"12345678901234567890"},"5":{"str":"92543e103926a42afd4c2644f7d793753239333f"},"6":{"i32":2},"7":{"lst":["str",3,"Host","X-Xiaomi-Timestamp","X-Xiaomi-Content-MD5"]}}
```

###6.2 常见传输层错误###
SDK相关的常见传输层错误对应的HTTP状态码如下：

```thrift
/**
 * HTTP状态码列表，用于传输层，签名错误等
 */
enum HttpStatusCode {
  /**
   * 请求格式错误，常见原因为请求参数错误导致服务端反序列化失败
   */
  BAD_REQUEST = 400,
  /**
   * 无效的认证信息，一般为签名错误
   */
  INVALID_AUTH = 401,
  /**
   * 客户端时钟不同步，服务端拒绝(为防止签名的重放攻击)
   */
  CLOCK_TOO_SKEWED = 412,
  /**
   * HTTP请求过大
   */
  REQUEST_TOO_LARGE = 413,
  /**
   * 内部错误
   */
  INTERNAL_ERROR = 500,
}
```

其中，常见的错误`CLOCK_TOO_SKEWED`和套接字超时错误的自动处理可参考现有的SDK实现。

7. 集群信息
----------------

目前结构化存储根据用户需求，包含了不同配置的三个集群，同时提供HTTP/HTTPS两种接入方式：

###7.1 在线集群###
  * **主集群** 
    * *集群名* - cnbj-s0.sds.api.xiaomi.com
    * *集群配置* - 后端采用高性能固态硬盘设备，为用户持续提供高吞吐率、低延迟的存储服务
    * *建议使用方式* - 适合随机读、对访问延迟要求敏感的应用
    
  * **备集群** 
    * *集群名* - cnbj-s0h0.sds.api.xiaomi.com
    * *集群配置* - 后端采用大容量磁盘设备，提供远程复制功能，为用户提供高可用和高可靠的存储服务
    * *建议使用方式* - 适合离线数据分析等数据**只读**的应用
  
###7.2 离线集群###
  * **集群名** - cnbj-h0.sds.api.xiaomi.com
  * **集群配置** - 后端采用大容量磁盘设备，为用户提供高带宽、低成本的存储服务
  * **建议使用方式** - 适合离线数据分析、批量顺序扫描、对延迟不敏感的应用
  
8. 附录
----------------

###8.1 常量与变量###

常量         | 示例
------------ | ---------
布尔常量     | true, false, [true], [false]
数值常量     | 1, 2e5, 10.0
字符串常量   | 'hello'
表属性       |  attr1, [attr2]

###8.2 操作符类表(按照优先级排列)###

操作符             |  说明           |  示例
------------------ | --------------- | --------
&#124;&#124;       |  字符串拼接     | 'hello ' &#124;&#124; 'world'
\* / %             |  乘，除，取余   |  1\*2
+ -                |  加，减         |  3-2.5
< <= > >= == != <> |  关系运算符     | 3 > 2
REGEXP             |  正则表达式匹配 | 'abc' regexp '[a-z]+'
NOT                |  取反           | not true
AND                |  与             | i > 3 and j > 5
OR                 |  或             |  a < 1 or a > 10
func(...)          |  函数调用       | rand(10)
(expr)             |  括号           |  (1 + 3) * 3
ISNULL             |  是否为null     |  i isnull
NOTNULL            |  是否不为null   |  j notnull

###8.3 函数列表###

目前支持的函数如下，如有新的需求，请联系技术支持。

函数                       |    说明
-------------------------- | --------------
string(numeric)            | 数值转换为字符串
lower(string)              | 转成小写字符
upper(string)              | 转成大写字符
length(string)             | 字符串长度
substr(string, int, int)   | 字串
trim(string)               | 去除字符串头尾空白符
max(numeric, numeric)      | 最大值
min(numeric, numeric)      | 最小值
abs(numeric)               | 绝对值
pow(numeric, numeric)      | 幂操作
log(numeric)               | 对数
rand(),rand(numeric)       | 随机数
now()                      | 当前UNIX时间戳，1970到现在的秒数

