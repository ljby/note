#### 应用场景

* 海量数据分析引擎
* 站内搜索引擎
* 数据仓库

### ES是什么

​        elasticsearch简写es，es是一个高扩展、开源的全文检索和分析引擎，它可以准实时地快速存储、搜索、分析海量的数据。

​        全文检索、结构化检索、数据分析

#### 什么是全文检索

​        全文检索是指计算机索引程序通过扫描文章中的每一个词，对每一 个词建立一个索引，指明该词在文章中出现的次数和位置，当用户查询时，检索程序就根据事先建立的索引进行查找，并将查找的结果反馈给用户的检索方式。这个过程类似于通过字典中的检索字表查字的过程。全文搜索搜索引擎数据库中的数据。

#### 倒排索引

<https://www.cnblogs.com/zlslch/p/6440114.html>

​        正向索引（forward index），反向索引（inverted index）更熟悉的名字是**倒排索引**。

​       在搜索引擎中每个文件都对应一个文件ID，文件内容被表示为一系列关键词的集合（实际上在搜索引擎索引库中，关键词也已经转换为关键词ID）。例如“文档1”经过分词，提取了20个关键词，每个关键词都会记录它在文档中的出现次数和出现位置。

​        得到正向索引的结构如下：

​        “文档1”的ID > 单词1：出现次数，出现位置列表；单词2：出现次数，出现位置列表；…………。

​        “文档2”的ID > 此文档出现的关键词列表。

　　一般是通过key，去找value。

​        当用户在主页上搜索关键词“华为手机”时，假设只存在正向索引（forward index），那么就需要扫描索引库中的所有文档，找出所有包含关键词“华为手机”的文档，再根据打分模型进行打分，排出名次后呈现给用户。因为互联网上收录在搜索引擎中的文档的数目是个天文数字，这样的索引结构根本无法满足实时返回排名结果的要求。

​        所以，搜索引擎会将正向索引重新构建为倒排索引，即把文件ID对应到关键词的映射转换为关键词到文件ID的映射，每个关键词都对应着一系列的文件，这些文件中都出现这个关键词。

​       得到倒排索引的结构如下：

​       “关键词1”：“文档1”的ID，“文档2”的ID，…………。

​       “关键词2”：带有此关键词的文档ID列表。

　  ==从词的关键字，去找文档。==

#### elasticsearch的作用

* 自动维护数据分布到多个节点的索引的建立，还有搜索请求分布到多个节点的执行
* 自动维护数据的冗余副本，保证一些机器宕机了，不会丢失任何数据
* 在lucene的基础上，封装了更多的高级功能，让我们快速的开发应用，开发更加复杂的应用：复杂的搜索功能、聚合分析的功能

#### elasticsearch的查询流程分析

<https://blog.csdn.net/u010454030/article/details/79794788>

​         我们都知道[es](https://www.baidu.com/s?wd=es&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)是一个分布式的存储和检索系统，在存储的时候默认是根据每条记录的_id字段做路由分发的，这意味着es服务端是准确知道每个document分布在那个shard上的。

​        相对比于CURD上操作，search一个比较复杂的执行模式，因为我们不知道那些document会被匹配到，任何一个shard上都有可能，所以一个search请求必须查询一个索引或多个索引里面的所有shard才能完整的查询到我们想要的结果。

​        找到所有匹配的结果是查询的第一步，来自多个shard上的数据集在分页返回到客户端的之前会被合并到一个排序后的list列表，由于需要经过一步取top N的操作，所以search需要进过两个阶段才能完成，分别是query和fetch。

##### （一）query（查询阶段）

​        当一个search请求发出的时候，这个query会被广播到索引里面的每一个shard（主shard或副本shard），每个shard会在本地执行查询请求后会生成一个命中文档的优先级队列。

​        这个队列是一个排序好的top N数据的列表，它的size等于from+size的和，也就是说如果你的from是10，size是10，那么这个队列的size就是20，所以这也是为什么深度分页不能用from+size这种方式，因为from越大，性能就越低。

​        es里面分布式search的查询流程如下：

1. 客户端发送一个search请求到Node 3上，然后Node 3会创建一个优先级队列它的大小=from+size
2. 接着Node 3转发这个search请求到索引里面每一个主shard或者副本shard上，每个shard会在本地查询然后添加结果到本地的排序好的优先级队列里面。
3. 每个shard返回docId和所有参与排序字段的值例如_score到优先级队列里面，然后再返回给coordinating节点也就是Node 3，然后Node3负责将所有shard里面的数据给合并到一个全局的排序的列表。

​        上面提到一个术语叫coordinating node，这个节点是当search请求随机负载的发送到一个节点上，然后这个节点就会成为一个coordinating node，它的职责是广播search请求到所有相关的shard上，然后合并他们的响应结果到一个全局的排序列表中然后进行第二个fetch阶段，注意这个结果集仅仅包含docId和所有排序的字段值，search请求可以被主shard或者副本shard处理，这也是为什么我们说增加副本的个数就能增加搜索吞吐量的原因，coordinating节点将会通过round-robin的方式自动负载均衡。

##### （二）fetch（读取阶段）

​        query阶段标识了那些文档满足了该次的search请求，但是我们仍然需要检索回document整条数据，这个阶段称为fetch

流程如下：

1. coordinating 节点标识了那些document需要被拉取出来，并发送一个批量的mutil get请求到相关的shard上
2. 每个shard加载相关document，如果需要他们将会被返回到coordinating 节点上
3. 一旦所有的document被拉取回来，coordinating节点将会返回结果集到客户端上。

​        这里需要注意，coordinating节点拉取的时候只拉取需要被拉取的数据，比如from=90，size=10，那么fetch只会读取需要被读取的10条数据，这10条数据可能在一个shard上，也可能在多个shard上所以 coordinating节点会构建一个multi-get请求并发送到每一个shard上，每个shard会根据需要从_source字段里面获取数据，一旦所有的数据返回，coordinating节点会组装数据进入单个response里面然后将其返回给最终的client。

##### 总结：

​        es的分布式search的查询流程分为query和fetch两个阶段，在query阶段会从所有的shard上读取相关document的docId及相关的排序字段值，并最终在coordinating节点上收集所有的结果数进入一个全局的排序列表后，然后获取根据from+size指定page页的数据，获取这些docId后再构建一个multi-get请求发送相关的shard上从_source里面获取需要加载的数据，最终再返回给client端，至此整个search请求流程执行完毕

### 基本概念

##### Cluster （集群）

​         一个集群包含一个或多个分配了相同的集群名称的节点。每个集群都有一个主节点是集群自动选择产生,并且可以决定如果当前主节点失败，哪些可以替换。

##### Document（文档）

​        文档是存储在elasticsearch中的一个==JSON文件==。这是相当与关系数据库中表的一行数据。每个文档被存储在索引中，并具有一个类型和一个id。一个文档是一个JSON对象（也被称为在其他语言中的 hash / hashmap / associative array（关联数组）），其中包含零个或多个字段 或者 键值对。原始JSON文档将被存储在索引的_source字段，在获得（getting）或者 搜索（searching）默认的返回时，得到或搜索文档。

##### Id（标识）

​        每个文档ID标识了一个文档。一个文档的索引/类型/ ID必须是唯一的。如果没有提供ID，将是自动生成。（还可以看到路由）

##### Field（字段）

​        文档中包含的一组字段或键值对。字段的值可以是一个简单的（标量）值（如字符串，整数，日期），或者一个嵌套的结构就像一个数组或对象。一个字段就是类似关系数据库表中的一列。映射的每个字段有一个字段的类型“type”（不要与文档类型混淆），表示那种类型的数据可以存储在该字段里，如：整数，字符串，对象。映射还允许你定义（除其他事项外）一个字段的值如何进行分析。

##### Index（索引）

​        索引就是像关系数据库中的“数据库”。通过映射可以定义成多种类型。索引是一个逻辑命名空间映射到一个或多个主要的分片，可以有零个或多个副本分片。

##### Mapping（映射）

​        映射是像关系数据库中的”模式定义“。每个索引都有一个映射，它定义了每个索引的类型，再加上一些索引范围的设置。映射可以被明确地定义，或者在一个文档被索引的时候自动生成。

##### Node（节点）

​         节点是属于elasticsearch群集的运行实例。测试的时候，在一台服务器可以启动多个节点，但通常情况下应该在一台服务器运行一个节点。在启动时，节点将使用单播（或组播，但是必须指定）来发现使用相同的群集名称的群集，并会尝试加入该群集。

##### Primary shard（主分片）

​         每个文档都存储在一个主要分片上。当你索引一个文档时，索引首先生成在主分片上，然后才到主分片的所有副本上。默认情况下，索引有5个主分片。您可以指定更多或更少的主分片来适应索引可以处理的文档数。一旦创建了索引，就不能改变索引中主分片的数量。

##### Replica shard（副本分片）

 每个主分片可以有零个或多个副本。副本是主分片的一个拷贝，有两个作用：

 1、故障转移：如果主分片有问题，副本分片可以提升为主分片；

 2、提高性能：获取和搜索请求可以处理主分片或副本分片。

 默认情况下，每个主分片有一个副本，不过索引的副本数量可以动态地改变。在同一个节点上，一个副本分片将永远不会和其主分片一起运行。

##### Routing（路由）

​       当你索引一个文档，它是存储在一个主分片里。这分片的选择是通过哈希的路由值。默认情况下，路由值来自文档的ID；如果该文档指定了父文档，则使用父文档的ID（以确保这个子文档和父文件都存储在相同的分片上）。这个路由值可以在索引的时候，通过指定数值或者配置字段映射来覆盖。

##### Shard（分片）

​        一个分片是一个单一的Lucene的实例。这是一个低级别的通过ElasticSearch自动管理的“工作者”单元。索引是一个逻辑命名空间指向主分片和副本分片。索引的主分片和副本分片的数量需要明确的指定。然而你的代码应该只处理一个索引。Elasticsearch分配集群中所有节点的分片。在节点出现故障或增加新节点的时候，可以自动的将一个节点上的分片移动到另一个节点上。

##### Source field（源字段）

​         默认情况下，你的JSON文档将被索引存储在_source字段里面，所有的get（获取）和search（搜索）请求将返回的该字段。这将允许你直接从搜索结果中访问到源数据，而不需要再次发起请求检索。

​         注：索引将返回完整的的JSON字符串给你，即使它包含无效的JSON。此字段里的内容不表示任何该对象里面的数据如何被索引。

##### Term（术语）

​        在elasticsearch里，术语(term)是一个被索引的精确值。术语 foo, Foo,FOO 是不想等的。术语（即精确值）可以使用“term”查询接口来查询。

##### Text(文本)

​        文本（或全文）是普通非结构化的文本，如本段。默认情况下，文本将被分析成术语，术语才是实际存储在索引中。文本字段在索引时需要进行分析，以便全文搜索，全文查询的关键字在搜索时，必须分析产生（搜索）与索引时相同的术语。

##### Type(类型)

​         Type是相当于关系数据库中的“表”。每种类型都有一列字段，用来定义文档的类型。映射定义了对在文档中的每个字段如何进行分析。

```java
//es         数据库
Document       行
Type           表  
Index          库
```



### elasticsearch的安装与部署

一.软件下载(百度网盘离线下载比较快,官网下载比较慢)

elasticsearch-6.5.4.ziphttps://pan.baidu.com/s/1ujmXbSCmcEpgnmyzZIGHhQ  提取码：bdzp 

logstash-6.5.4.ziphttps://pan.baidu.com/s/1tBkCrP1RcDhVJ6cnsOJaeg     提取码：oimw 

kibana-6.5.4.ziphttps://pan.baidu.com/s/1OuDSpb0DSK49KkU_X5FTwQ  提取码：v9jw 

#### elasticsearch（window版）

```java
//解压后，cmd进入elasticsearch的bin目录下
elasticsearch.bat
//检验是否启动成功，进入浏览器
http://localhost:9200/?pretty
//修改集群名称
elasticsearch.yml----->cluster.name：集群名称，默认则会自动分配 
//下载kibana，启动kibana，进入bin
kibana.bat
端口号：5601
进入dev Tools界面
```

#### elasticsearch和logstash和kibana

#####   ELK是什么？

​        ELK Stack是软件集合Elasticsearch、Logstash、Kibana的简称，由这三个软件及其相关的组件可以打造大规模日志实时处理系统。

​        Elasticsearch 是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制， restful 风格接口，多数据源，自动搜索负载等。

​        Logstash 是一个完全开源的工具，他可以对你的日志进行收集、分析，并将其存储供以后使用（如，搜索）。

​         kibana 也是一个开源和免费的工具，他 Kibana 可以为 Logstash 和 ElasticSearch 提供的日志分析友好的 Web 界面，可以帮助您汇总、分析和搜索重要数据日志 

​         ogstash 工作的三个阶段： input、filter、output

#### elasticsearch和logstash和kibana的安装

##### Elasticsearch部署

解压之后，首先进行配置。配置文件路径：/config/elasticsearch.yml。 

主要配置： 

\- cluster.name：集群名称，默认则会自动分配 

\- node.name是节点名称，默认则会自动分配 

\- http.port是访问端口，默认9200。

我的配置如下：

cluster.name: elasticsearch

node.name: node-1

执行以下语句即可启动Elasticsearch。其中，-d是通过守护进程启动。

./bin/elasticsearch

./bin/elasticsearch -d

检测是否启动成功：

curl 'http://localhost:9200/?pretty'

或者直接在浏览器访问这个地址，就可以显示类似如下的返回结果：

```java
{
  "name" : "node-1",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "Q79rR5mCTICI7UfEcwmDnA",
  "version" : {
    "number" : "5.0.1",
    "build_hash" : "080bb47",
    "build_date" : "2016-11-11T22:08:49.812Z",
    "build_snapshot" : false,
    "lucene_version" : "6.2.1"
  },
  "tagline" : "You Know, for Search"
}
```

这样，我们的Elasticsearch就成功启动了。

##### kibana部署

​        使用其开发界面，进行操作elasticsearch，解压kibana之后，首先进行配置，配置文件路径：config/kibana.yml 

主要配置：

```java
server.host：kibana的访问ip地址
server.port：kibana的访问端口，默认5601
elasticsearch.url：对应的Elasticsearch地址
kibana.index：kibana的index索引名

我的配置如下：
server.host: "localhost"
elasticsearch.url: "http://localhost:9200"
kibana.index: ".kibana"
server.port: 5601
    
启动方式和Elasticsearch一样：
./bin/kibana
浏览器访问http://localhost:9200即可进入kibana首页。
```

##### logstash部署

​        解压logstash之后，首先进行配置（怎么还是这句话……）。哦不能首先配置，我们得先来了解一下logstash的工作原理和具体配置。

**工作原理**

Logstash使用管道方式进行日志的搜集处理和输出，从输入到输出，整个过程包括了三个阶段：输入input –> 处理filter（不是必须的） –> 输出output，每个阶段都可以选择很多的插件配合工作，比如file、elasticsearch、redis；也可以指定多种方式，比如输出到Elasticsearch或者输出控制台。

**配置格式**

```java
配置格式要包括整个阶段的三部分，所以标准格式如下：
input {...} 
filter {...} 
output {...}
每一部分可以指定多个方式，比如输入要从控制台和数据库输入，按照如下配置：
input {
    stdin {
    }
    jdbc {
        ...
    }
}
输出如果是输出控制台或者Elasticsearch，按照如下配置：
output {
  elasticsearch {
  hosts => "localhost:9200"
  index => "mysql_accept"
  document_type => "accept"
  }
  stdout {
  codec => json_lines
  }
}
```

因为filter的配置不是必须的，所以这里暂时不表，等后续需要再讲解。

**常用命令**

-f指定Logstash的配置文件，根据配置文件配置logstash并进行启动：

./logstash -f logstash.conf

-e字符串当做配置启动,”“则默认控制台输入控制台输出

./logstash -e ""

-t测试配置文件是否正确并退出

./logstash -t logstash.conf

**启动**

上文已经讲解了logstash的基本配置以及启动命令。不过如果我们的logstash只是做从控制台输入再从控制台输出的工作实在是太没有意义了。一般的业务需求中，是需要将海量的数据从数据库中索引到Elasticsearch中的，所以这里主要介绍一下如何从MySQL和oracle将数据保存到Elasticsearch。Jdbc input plugin是官方文档对于jdbc插件的介绍，感兴趣的话可以看一看，毕竟官方文档是最权威的。 

jdbc的input配置中，以下几项是比较重要的：

jdbc_driver_library：jdbc的依赖包路径

jdbc_driver_class：注册jdbc驱动。这里要注意！如果是MySQL则配置为”com.mysql.jdbc.Driver”，如果是oracle则配置为”Java::oracle.jdbc.driver.OracleDriver”（这里笔者找了半天的问题，后来发现官方文档里其实有写）

jdbc_connection_string：数据库连接

jdbc_user：数据库用户名

jdbc_password：数据库密码

schedule：定时服务配置，多长时间执行一次数据同步，格式为* * * * *，对应分 时 天 月 年。

statement：每次同步时的sql执行语句

statement_filepath：如果sql语句写在文件中，则在这里配置sql文件的路径

以下几项是增量更新的配置。logstash对于数据库的增两个更新有两种控制方式，一种是根据时间，一种是根据自增判断。执行完一次同步后，logstash会将最后同步的那条记录的时间或者自增值记录到文件中，下次同步时则从最新记录那里开始执行，记录值在sql语句中用:sql_last_value作为替代。 

如果是用时间做控制，则sql语句这样写：

select * from tablename where update_time>:sql_last_value

如果不配置，则每次都会全量更新，这样比较浪费资源，所以建议配置：

last_run_metadata_path：记录最后更新值的保存路径。

use_column_value：true或者false。如果是true则使用自增值控制，false则使用时间控制。

tracking_column：选择控制增量更新的字段，对于上面的例子，这里配置成update_time

对于output到Elasticsearch的配置，最好是配置好数据索引到Elasticsearch的index、type和文档的id。下面是一个简单的配置：

```java
output {
  elasticsearch {
  hosts => "localhost:9200"
  index => "my_index"
  document_type => "my_type"
  document_id => "%{id}"
  }
  stdout {
  codec => json_lines
  }
}
```

​        其中，index为同步数据时要保存到的索引，document_type为要保存到的type，document_id则配置保存成单个文档的id，%{id}中的id对应sql语句中的主键id字段名，同时数据也会输出到控制台。 

下面是我的一个配置：  (6.5.4)

```java
input {
  jdbc {
    jdbc_driver_library => "D:\search\logstash-6.5.4\bin\mysql-driver\mysql-connector-java-5.1.47.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/abc"
    jdbc_user => "root"
    jdbc_password => "123456"
    schedule => "* * * * *"
    jdbc_default_timezone => "Asia/Shanghai"
    statement => "select * from userinfo"
    \#statement_filepath => "D:\search\logstash-6.5.4\bin\mysql-driver/jdbc.sql"
    use_column_value  => false
    last_run_metadata_path => "D:\search\logstash-6.5.4/last_run.txt"
      }
}
output {
    elasticsearch {
        hosts => ["127.0.0.1:9200"]
        index => "abc"
        document_id => "%{userid}"
        document_type => "userinfo"
    }
    stdout {
        codec => json_lines
    }
}
```

执行以下命令，就可以将数据同步到之前配置好的Elasticsearch中了：

./logstash -f logstash.conf              (执行配置的conf文件)

如果出现错误   先检查配置文件是否足够     然后一直查相应版本的conf文件是否错误

### elatisearch的基本使用

#### 简单的集群管理

​          es提供了一套api，即==cat api==，可以查看es中各种各样的数据

```java
//快速检查集群的健康状况
GET /_cat/health?v
//健康状态:status
green：每个索引的primary shard和replica shard都是active状态的
yellow：每个索引的primary shard都是active状态的，但是部分replica shard处于不可用状态
red：不是所有的primary shard都是active状态的，部分索引有数据丢失

//快速查看集群中有哪些索引
GET /_cat/indices?v

//简单的索引操作
创建索引：PUT /test_index?pretty
删除索引：DELETE /test_index?pretty

//新增商品：新增文档，建立索引
PUT /index/type/1
PUT /索引名/product/1{
    "name":"www"
    "desc":"com"
    "price":30
     "tags"：["http","https"]
}//可以插多次  
   es会自动建立index和type，不需要提前创建，而且es会对document每个field都建立倒排索引，让其可以被搜索
   
//检索文档
GET /index/type/id
》》get /ecommerce/product/1

//替换文档（必须带上所有的field）
get /ecommerce/product/1  

//更新文档    
POST /ecommerce/product/1/_update   
{ "doc": {
   "name"：""
}   }

//删除文档
DELETE /ecommerce/product/1
```

```java
搜索参数:
took: 耗费时间
timedout:  是否超时
shard:分片
hits.total:查询结果数量
max_score:匹配分数
index: 索引    数据库
document-type:  表
document-id  :主键

1.query  string  search http请求参数
2.query  DSL  特定领域的语言    es
查询所有商品 
get  /index/type/_search
{
   "query":{"match_all":{}}
} 
按字段查询  
GET /abc/userinfo/_search     //索引   type    /数据库  表
{
  "query":{"match": {           //查询语句  mactch 抓取关键词   在此抓了name   此处是json的语法
    "username": "小绿折"      
  }},
  "sort": [                           //排序
    {
      "userage": {               //选择列名   数字可以按大小排   字符按列名.keyword排
        "order": "desc"        
      }
    }
  ]
}
3.分页查询
GET /abc/userinfo/_search
{
  "query": {"match_all": {}}
  , "from": 0       //起始
    , "size": 2     //每页显示
}
4.按想要显示的内容查询    -source  相当于列  里面存放所有的列
GET /abc/userinfo/_search
{
  "query": {"match_all": {}},
  "_source": ["useraddress","userage","username"]     //添加限制  只显示这三个
}
5.query  filter    查询拦截器（对数据进行过滤）
GET /abc/userinfo/_search
{
  "query": {
    "bool": {            //多选
       "must": [        //必须满足
         {"match": {       //限制
           "username": "小"
         }}
       ]
      , "filter": {         //拦截器
         "range": {      //范围
           "userage": {
             "gte": 1,
             "lte": 25      
           }
         }
      }
    }
    
  }
}
6.全文查询
GET /abc/userinfo/_search
{
  "query": {
    "match": {
      "useraddress": " 罗  西"     //模糊查询  会搜索出所有带有 罗  西  的地点   按照搜索字的顺序展示   按匹配度
    }
  }
}
7.phrase  search  短语搜索
GET /abc/userinfo/_search
{
  "query": {
    "match_phrase": {        //精确查询  有一个字 就查包含这个字的数据   俩个字  但是查不到就不会查到
      "useraddress": " 罗西"
    }
  }
}
8.高亮查询
GET /abc/userinfo/_search
{
  "query": {
    "match": {
      "useraddress": " 罗"
    }
  }
  , "highlight": {        //查询到包含罗字段的数据后 会把罗 高亮显示  在页面中
        "fields": {   
          "罗":{}
        }
  }
}
二.综合查询
GET /abc/userinfo/_search
{
  "query": {
    "multi_match": {
    "query": "唱歌",
     "fields":["intersts","name"]
      }
}
}

GET /abc/userinfo/_search
{
  "query": {
    "match_phrase": {
    "interests":"锻炼,歌唱"
      }
 }
}
```

```java
//"aggs"聚合分析：
//对名称中包含xxx的商品，计算每个tag下的商品数量
GET /ecommerce/product/_search
{
   "size":0
   "query": {
     "match":{ 
          "name":"xxx"
     }
   },
   "aggs":{
      "all_tags":{
         "terms":{
            "field":"tags"
         }
      }
   }
}

//先分组，后计算平均值
GET /ecommerce/product/_search
{
   "size":0
   "aggs":{
      "group_by_tags":{
         "terms":{
            "field":"tags"
         },
         "aggs":{
             "avg_price":{
                 "avg":{
                     "field":"price"
                 }
             }
         }
      }
   }
}

//在上面基础上，进行排序
{
   "size":0
   "aggs":{
      "group_by_tags":{
         "terms":{
            "field":"tags"
            "order":{
                "avg_price":"desc"   //降序排序
            }
         },
         "aggs":{
             "avg_price":{
                 "avg":{
                     "field":"price"
                 }
             }
         }
      }
   }
}
```

