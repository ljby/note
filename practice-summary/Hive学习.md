         Hive 依赖于 HDFS 存储数据，Hive 将 HQL 转换成 MapReduce 执行，所以说 Hive 是基于 Hadoop 的一个数据仓库工具，实质就是一款基于 HDFS 的 MapReduce 计算框架，对存储在 HDFS 中的数据进行分析和管理

​          hdfs 是提供数据存储的，mapreduce 是方便数据计算的

学习链接：<https://www.cnblogs.com/qingyunzong/p/8707885.html>

#### HIVE基础

##### 概念

* Hive 由 Facebook 实现并开源

* 是基于 Hadoop 的一个数据仓库工具

* 可以将结构化的数据映射为一张数据库表

* 并提供 HQL(Hive SQL)查询功能

* 底层数据是存储在 HDFS 上

* Hive的本质是将 SQL 语句转换为 MapReduce 任务运行

* 使不熟悉 MapReduce 的用户很方便地利用 HQL 处理和计算 HDFS 上的结构化的数据，适用于离线的批量数据计算。

##### HIVE架构

![1228818-20180403193501903-1989526977](images\1228818-20180403193501903-1989526977.png)

hive的内部架构由四部分组成：

* ==用户接口==: shell/CLI, jdbc/odbc, webui Command Line Interface
  　　CLI，Shell 终端命令行（Command Line Interface），采用交互形式使用 Hive 命令行与 Hive 进行交互，最常用（学习，调试，生产）

　　JDBC/ODBC，是 Hive 的基于 JDBC 操作提供的客户端，用户（开发员，运维人员）通过 这连接至 Hive server 服务

　　Web UI，通过浏览器访问 Hive

* ==跨语言服务== ： thrift server 提供了一种能力，让用户可以使用多种不同的语言来操纵hive
  　　Thrift 是 Facebook 开发的一个软件框架，可以用来进行可扩展且跨语言的服务的开发， Hive 集成了该服务，能让不同的编程语言调用 Hive 的接口

* ==底层的Driver==： 驱动器Driver，编译器Compiler，优化器Optimizer，执行器Executor
  　　Driver 组件完成 HQL 查询语句从词法分析，语法分析，编译，优化，以及生成逻辑执行 计划的生成。生成的逻辑执行计划存储在 HDFS 中，并随后由 MapReduce 调用执行

　　**Hive 的核心是驱动引擎**， 驱动引擎由四部分组成：

```java
(1) 解释器：解释器的作用是将 HiveSQL 语句转换为抽象语法树（AST）
(2) 编译器：编译器是将语法树编译为逻辑执行计划
(3) 优化器：优化器是对逻辑执行计划进行优化
(4) 执行器：执行器是调用底层的运行框架执行逻辑执行计划
```

* ==元数据存储系统== ： RDBMS MySQL
  　　元数据，通俗的讲，就是存储在 Hive 中的数据的描述信息。

　　Hive 中的元数据通常包括：表的名字，表的列和分区及其属性，表的属性（内部表和 外部表），表的数据所在目录

　　Metastore 默认存在自带的 Derby 数据库中。缺点就是不适合多用户操作，并且数据存 储目录不固定。数据库跟着 Hive 走，极度不方便管理

　　解决方案：通常存我们自己创建的 MySQL 库（本地 或 远程）

　　Hive 和 MySQL 之间通过 MetaStore 服务交互

##### Hive数据组织

* Hive 的存储结构包括数据库、表、视图、分区和表数据等。数据库，表，分区等等都对 应 HDFS 上的一个目录。表数据对应 HDFS 对应目录下的文件。

* Hive 中所有的数据都存储在 HDFS 中，没有专门的数据存储格式，因为 Hive 是读模式 （Schema On Read），可支持 TextFile，SequenceFile，RCFile 或者自定义格式等

* 只需要在创建表的时候告诉 Hive 数据中的列分隔符和行分隔符，Hive 就可以解析数据

　　Hive 的默认列分隔符：控制符 Ctrl + A，\x01 Hive 的

　　Hive 的默认行分隔符：换行符 \n

* Hive 中包含以下数据模型：

　　database：在 HDFS 中表现为${hive.metastore.warehouse.dir}目录下一个文件夹

　　table：在 HDFS 中表现所属 database 目录下一个文件夹

　　external table：与 table 类似，不过其数据存放位置可以指定任意 HDFS 目录路径

　　partition：在 HDFS 中表现为 table 目录下的子目录

　　bucket：在 HDFS 中表现为同一个表目录或者分区目录下根据某个字段的值进行 hash 散 列之后的多个文件

　　view：与传统数据库类似，只读，基于基本表创建

#### HIVE安装步骤及初始化（linux）

##### 下载地址：

##### 先安装mysql

MySql安装过程http://www.cnblogs.com/qingyunzong/p/8294876.html

##### 解压安装包

```shell
tar -zxvf apache-hive-2.3.3-bin.tar.gz -C apps/
```

##### 修改配置文件

```java
//配置文件所在目录apache-hive-2.3.3-bin/conf
/home/hadoop/apps/apache-hive-2.3.3-bin/conf

//新建hive-site.xml并添加以下内容
vi hive-site.xml 

<configuration>
        <property>
                <name>javax.jdo.option.ConnectionURL</name>
                <value>jdbc:mysql://hadoop1:3306/hivedb?createDatabaseIfNotExist=true</value>
                <description>JDBC connect string for a JDBC metastore</description>
                <!-- 如果 mysql 和 hive 在同一个服务器节点，那么请更改 hadoop1 为 localhost -->
        </property>
        <property>
                <name>javax.jdo.option.ConnectionDriverName</name>
                <value>com.mysql.jdbc.Driver</value>
                <description>Driver class name for a JDBC metastore</description>
        </property>
        <property>
                <name>javax.jdo.option.ConnectionUserName</name>
                <value>root</value>
                <description>username to use against metastore database</description>
        </property>
        <property>
                <name>javax.jdo.option.ConnectionPassword</name>
                <value>root</value>
        <description>password to use against metastore database</description>
        </property>
</configuration>

//可选配置，该配置信息用来指定 Hive 数据仓库的数据存储在 HDFS 上的目录
 <property>
                <name>hive.metastore.warehouse.dir</name>
                <value>/hive/warehouse</value>
                <description>hive default warehouse, if nessecory, change it</description>
        </property>  
```

##### 加入 MySQL 驱动包（mysql-connector-java-5.1.40-bin.jar）该 jar 包放置在 hive 的根路径下的 lib 目录

##### 安装完成，配置环境变量

```java
//lib路径下配置
 vi ~/.bashrc 
//内容
#Hive
export HIVE_HOME=/home/hadoop/apps/apache-hive-2.3.3-bin
export PATH=$PATH:$HIVE_HOME/bin
//立即生效
source ~/.bashrc
```

##### 验证Hive的安装

```shell
hive --help
```

##### 初始化元数据库

```java
//当使用的 hive 是 2.x 之前的版本，不做初始化也是 OK 的，当 hive 第一次启动的 时候会自动进行初始化，只不过会不会生成足够多的元数据库中的表。在使用过程中会 慢慢生成。但最后进行初始化。如果使用的 2.x 版本的 Hive，那么就必须手动初始化元 数据库。使用命令
schematool -dbType mysql -initSchema
```

##### 启动Hive客户端

```shell
hive --service cli
```

#### HIVE基本使用

##### 创建一个数据库

```
create database myhive;
```

##### 使用新的数据库

```java
 use myhive;
```

##### 切换数据库

```java
use database_name;
```

##### 查看当前使用的数据库

```java
select current_database();
```

##### 创建表

```java
create table student(id int, name string, sex string, age int, department string) row format delimited fields terminated by ",";
```

##### 增加数据

```java
//文件student.txt
hive> load data local inpath "/home/hadoop/student.txt" into table student;
```

##### 查询数据

```java
select * from student;
```

##### 查看表结构

```java
desc student; //查看表结构
desc extended student;
desc formatted student; //.查看表详细属性
```

