```java
    hdfs 对应 namenode 和 datanode. namenode 负责保存元数据的基本信息， datanode 直接存放数据本身，整个HDFS集群由==Namenode和Datanode构成master-worker（主从）模式==
```

#### HA

​      系统对外正常提供服务时间的百分比

​      无法正常提供服务：NameNode的升级与维护、异常(系统崩溃、hdfs发生故障)

#### HA方案

​        主要解决NameNode的单点故障问题，即使Secondary NameNode有利于保护数据丢失，但并不能提高高可用性

​        采用HA的HDFS集群配置两个NameNode，分别处于Active和Standby状态。当Active NameNode故障之后，Standby接过责任继续提供服务，用户没有明显的中断感觉，一般耗时在几十秒到数分钟

- 主备需要共享edit log存储，Standby通过回放edit log同步数据

- DataNode需要同时往主备发送Block Report

  ​         因为Block映射数据存储在内存中（不是在磁盘上），为了在Active NameNode挂掉之后，新的NameNode能够快速启动，不需要等待来自Datanode的Block Report，DataNode需要同时向主备两个NameNode发送Block Report。

- 客户端需要配置failover模式（对用户透明） 
         Namenode的切换对客户端来说是无感知的，通过客户端库来实现。客户端在配置文件中使用的HDFS URI是逻辑路径，映射到一对Namenode地址。客户端会不断尝试每一个Namenode地址直到成功。

- Standby替代Secondary NameNode 
         如果没有启用HA，HDFS独立运行一个守护进程作为Secondary Namenode。定期checkpoint，合并镜像文件和edit日志。如果当主Namenode失败时，备份Namenode正在关机（停止 Standby），运维人员依然可以从头启动备份Namenode，这样比没有HA的时候更省事，算是一种改进，因为重启整个过程已经标准化到Hadoop内部，无需运维进行复杂的切换操作。NameNode的切换通过代failover controller来实现。failover controller有多种实现，默认实现使用ZooKeeper来保证只有一个Namenode处于active状态

#### 背景

​         单机容量往往无法存储大量数据，需要跨机器存储。**统一管理分布在集群上的文件系统**称为分布式文件系统。而一旦在系统中，引入网络，就不可避免地引入了所有网络编程的复杂性，例如挑战之一是如果保证在节点不可用的时候数据不丢失。

​         传统的网络文件系统（NFS）虽然也称为分布式文件系统，但是其存在一些限制。由于NFS中，文件是存储在单机上，因此无法提供可靠性保证，当很多客户端同时访问NFS Server时，很容易造成服务器压力，造成性能瓶颈。另外如果要对NFS中的文件进行操作，需要首先同步到本地，这些修改在同步到服务端之前，其他客户端是不可见的。某种程度上，NFS不是一种典型的分布式系统，虽然它的文件的确放在远端（单一）的服务器上面

#### HDFS设计原则

- 存储非常大的文件

  ​        这里非常大指的是几百M、G、或者TB级别。实际应用中已有很多集群存储的数据达到PB级别。根据Hadoop官网，Yahoo！的Hadoop集群约有10万颗CPU，运行在4万个机器节点上。更多世界上的Hadoop集群使用情况，参考Hadoop官网

- 采用流式的数据访问方式:

  ​        HDFS基于这样的一个假设：最有效的数据处理模式是一次写入、多次读取数据集经常从数据源生成或者拷贝一次，然后在其上做很多分析工作 分析工作经常读取其中的大部分数据，即使不是全部。 因此读取整个数据集所需时间比读取第一条记录的延时更重要

- 运行于商业硬件上:

  ​        Hadoop不需要特别贵的、reliable的机器，可运行于普通商用机器（可以从多家供应商采购） 商用机器不代表低端机器在集群中（尤其是大的集群），节点失败率是比较高的HDFS的目标是确保集群在节点失败的时候不会让用户感觉到明显的中断

#### HDFS核心

##### 元数据

​        即维护HDFS文件系统中的文件和目录所需要的信息

* 从形式上讲，元数据可分为==内存元数据==和==元数据文件==两种。其中 NameNode需要注意的是，具体的文件内容不是元数据，元数据是用于描述和组织具体的文件内容，如果没有元数据，具体的文件内容将变得没有意义。元数据的作用十分重要，它的可用性直接决定了 HDFS 的可用性。在内存中维护整个文件系统的元数据镜像，用于 HDFS 的管理；元数据文件则用于持久化存储。
* 从类型上讲，元数据有三类重要信息：
  * 第 1 类是文件和目录自身的属性信息，例如文件名、目录名、父目录信
    息、文件大小、创建时间、修改时间等
  *  第 2 类记录文件内容存储相关信息，例如文件分块情况、副本个数、每
    个副本所在的 Data Node 信息等
  * 第 3 类用来记录 HDFS 中所有 Data Node 的信息，用于 Data Node 管理。
* 从来源上讲，元数据主要来源于 NameNode 磁盘上的元数据文件（它包括元数据镜像 fsimage 和元数据操作日志 edits 两个文件）以及各个 Data Node 的上报信息

##### client

​         Client（代表用 户） 通过与 NameNode 和 DataNode 交互访问 HDFS 中 的文件， Client 提供了一个类似 POSIX 的文件系统接口供用户调用

##### Namenode

​         整个 Hadoop 集群中只有一个 NameNode。 它是整个系统的“ 总管”， 负责管理 HDFS 的目 录树和相关的文件元数据信息。 这些信息是以“ **fsimage**”（ HDFS 元数据镜像文件）和 “ **editlog**”（HDFS 文件改动日志）两个文件形式存放在本地磁盘，当 HDFS 重启时重新构造出 来的。此外， NameNode 还负责监控各个 DataNode 的健康状态， 一旦发现某个 DataNode 宕 掉，则将DataNode 移出 HDFS 并重新备份其上面的数据

##### Secondary NameNode

​         Secondary NameNode 最重要的任务并不是为 NameNode 元数据进行热备份， 而是定期合并 fsimage 和 edits 日志， 并传输给 NameNode。 这里需要注意的是，为了减小 NameNode 压 力， NameNode 自己并不会合并 fsimage 和 edits， 并将文件存储到磁盘上， 而是交由 Secondary NameNode 完成，它会维护一个合并后的namespace image副本， 可用于在Namenode完全崩溃时恢复数据

​         因为合并操作需要耗费大量的CPU和内存。其数据落后于Namenode，因此当Namenode完全崩溃时，会出现数据丢失。 通常做法是拷贝NFS中的备份元数据到Second，将其作为新的主Namenode

##### DataNode

​         一般而言， 每个 Slave 节点上安装一个 DataNode， 它负责实际的数据存储， 并将数据信息定期 汇报给 NameNode。 DataNode 以固定大小的 block 为基本单位组织文件内容， 默认情况下 block 大小为 64MB。 当用户上传一个大的文件到 HDFS 上时， 该文件会被切分成若干个 block， 分别存储到不同的 DataNode ； 同时，为了保证数据可靠， 会将同一个 block 以流水线方式写到13/04/2018 Page 260 of 283 若干个（默认是 3，该参数可配置）不同的 DataNode 上。 这种文件切割后存储的过程是对用户 透明的

##### Blocks

​         HDFS的Block块比一般单机文件系统大得多，默认为128M。HDFS的文件被拆分成block-sized的chunk，chunk作为独立单元存储。比Block小的文件不会占用整个Block，只会占据实际大小。例如， 如果一个文件大小为1M，则在HDFS中只会占用1M的空间，而不是128M

​         Block设计这么大，是为了最小化查找（seek）时间，控制定位文件与传输文件所用的时间比例

​         Block的拆分使得单个文件大小可以大于整个磁盘的容量，构成文件的Block可以分布在整个集群， 理论上，单个文件可以占据集群中所有机器的磁盘。；Block的抽象也简化了存储系统，对于Block，无需关注其权限，所有者等内容（这些内容都在文件级别上进行控制）； Block作为容错和高可用机制中的副本单元，即以Block为单位进行复制

#### HDFS读取文件

```java
对HDFS的大多数操作还是通过 FileSystem 来操作，HDFS的实现类 DistributedFileSystem 及相关类
```

其原理，主要包括以下几个步骤：

- 首先调用FileSystem对象的==open==方法，其实获取的是一个==DistributedFileSystem==的实例。
- DistributedFileSystem通过**RPC**(远程过程调用)获得文件的第一批block的locations，同一**block**按照重复数会返回多个locations，这些locations按照hadoop拓扑结构排序，距离客户端近的排在前面。
- 前两步会返回一个FSDataInputStream对象，该对象会被封装成 **DFSInputStream**对象，DFSInputStream可以方便的管理datanode和namenode数据流。客户端调用read方法，DFSInputStream就会找出离客户端最近的datanode并连接datanode。
- 数据从datanode源源不断的流向客户端。
- 如果第一个block块的数据读完了，就会关闭指向第一个block块的datanode连接，接着读取下一个block块。这些操作对客户端来说是透明的，从客户端的角度来看只是读一个持续不断的流。
- 如果第一批block都读完了，DFSInputStream就会去namenode拿下一批blocks的location，然后继续读，如果所有的block块都读完，这时就会关闭掉所有的流。

```java
//获取FileSystem实例
public static FileSystem get(Configuration conf) throws IOException
public static FileSystem get(URI uri , Configuration conf) throws IOException
public static FileSystem get(URI uri , Configuration conf，String user) throws IOException

//如果是本地文件，通过LocalFileSystem获取本地文件系统
public static LocalFileSystem getLocal(COnfiguration conf) throws IOException

//调用FileSystem的open方法获取一个输入流
public FSDataInputStream open(Path f) throws IOException
public abstarct FSDataInputStream open(Path f , int bufferSize) throws IOException

//进行数据操作
public class FSDataInputStream extends DataInputStream
    implements Seekable, PositionedReadable, 
      ByteBufferReadable, HasFileDescriptor, CanSetDropBehind, CanSetReadahead,
      HasEnhancedByteBufferAccess
      
//随机读取操作通过Seekable接口定义
public interface Seekable {
    void seek(long pos) throws IOException;  //seek操作开销大
    long getPos() throws IOException;
}

//也可部分读取
public interface PositionedReadable{
    public int read(long pistion ,byte[] buffer,int offser , int length) throws IOException;
    public int readFully(long pistion ,byte[] buffer,int offser , int length) throws IOException;
    public int readFully(long pistion ,byte[] buffer) throws IOException;
}
```

```java
//读取文件
Configuration conf = new Configuration();
FileSystem fs = FileSystem.get(conf);
Path fileName = new Path("hdfs://localhost:8088/user/hadoop/test.txt");
FSDDataInputStream is = fs.open(fileName);
BufferedReader rd = new BufferedReader(new InputStreamReader(is));
String s = rd.readLine();
//此处输出
rd.close();
fs.close();//关闭hdfs
```

​             **Configuration**是hadoop中五大组件的公用类，所以放在了core下，org.apache.hadoop.conf.Configruration。这个类是作业的配置信息类，任何作用的配置信息必须通过Configuration传递，因为通过Configuration可以实现在多个mapper和多个reducer任务之间共享信息

#### HDFS写入文件

其原理，主要包括以下几个步骤：

- 客户端通过调用 DistributedFileSystem 的==create==方法，创建一个新的文件。
- DistributedFileSystem 通过 RPC（远程过程调用）调用 NameNode，去创建一个没有blocks关联的新文件。创建前，NameNode 会做各种校验，比如文件是否存在，客户端有无权限去创建等。如果校验通过，NameNode 就会记录下新文件，否则就会抛出IO异常。
- 前两步结束后会返回 FSDataOutputStream 的对象，和读文件的时候相似，FSDataOutputStream 被封装成 DFSOutputStream，DFSOutputStream 可以协调 NameNode和 DataNode。客户端开始写数据到DFSOutputStream,DFSOutputStream会把数据切成一个个小**packet**，然后排成队列 data queue。
- DataStreamer 会去处理接受 data queue，它先问询 NameNode 这个新的 block 最适合存储的在哪几个DataNode里，比如重复数是3，那么就找到3个最适合的 DataNode，把它们排成一个 pipeline。DataStreamer 把 packet 按队列输出到管道的第一个 DataNode 中，第一个 DataNode又把 packet 输出到第二个 DataNode 中，以此类推。
- DFSOutputStream 还有一个队列叫 ack queue，也是由 packet 组成，等待DataNode的收到响应，当pipeline中的所有DataNode都表示已经收到的时候，这时akc queue才会把对应的packet包移除掉。
- 客户端完成写数据后，调用close方法关闭写入流。
- DataStreamer 把剩余的包都刷到 pipeline 里，然后等待 ack 信息，收到最后一个 ack 后，通知 DataNode 把文件标示为已完成。

```java
String localSrc =  args[0];
String dst = args[1];
 
InputStream in = new BufferedInputStream(new FileInputStream(localSrc));
 
Configuration conf = new Configuration();
FileSystem fs = FileSystem.get(URI.create(dst),conf);
 
OutputStream out = fs.create(new Path(dst), new Progressable(){
    public vid progress(){
        System.out.print(...);
    }
});
 
IOUtils.copyBytes(in , out, 4096,true);
```

#### 目录操作

​        使用==mkdirs==()方法,会自动创建没有的上级目录；

​        HDFS中元数据封装在FileStatus类中，包括长度、block size，replicaions，修改时间、所有者、权限等信息。使用FileSystem提供的==getFileStatus==方法获取**FileStatus**

​         exists()方法判断文件或者目录是否存在；

```java
//查看目录或文件的信息
  public abstract FileStatus[] listStatus(Path f) throws FileNotFoundException, 
                                                         IOException;
```

​          path是个文件的时候，返回长度为1的数组。FileUtil提供的stat2Paths方法用于将FileStatus转化为Path对象

globStatus则使用通配符对文件路径进行匹配：

```java
public FileStatus[] globStatus(Path pathPattern) throws IOException
```

​          PathFilter用于自定义文件名过滤，不能根据文件属性进行过滤，类似于java.io.FileFilter。例如下面这个例子排除到给定正则表达式的文件：

```java
public interfacePathFilter{   boolean accept(Path path);   }
```

#### 删除数据

​        使用FileSystem的==delete==()方法

```java
public boolean delete(Path f , boolean recursive) throws IOException;
```

#### hadoop fs命令行交互

```java
hadoop fs -copyFromLocal // 从本地复制文件到HDFS
hadoop fs mkdir // 创建目录
hadoop fs -ls  // 显示指定的文件的详细信息 
hadoop hfs //只能适用于HDFS文件系统 
```

#### Linux下操作HDFS

##### 启动HDFS

首先，格式化配置HDFS文件系统，打开NameNode(HDFS服务器)，然后执行以下命令

```
$ hadoop namenode -format 
```

 	格式化HDFS后，启动分布式文件系统，以下命令将启动名称节点和数据节点的集群

```
$ start-dfs.sh 
```

##### HDFS的文件列表

 	   加载服务器信息后，使用'ls' 可以找出文件列表中的目录，文件状态，下面给出的是ls，可以传递一个目录或文件名作为参数的语法

```
$ $HADOOP_HOME/bin/hadoop fs -ls <args>
```

##### 将数据插入到HDFS

 	   假设在本地系统，这是所谓的file.txt文件中的数据,应当保存在HDFS文件系统。按照下面给出插入在Hadoop的文件系统所需要的文件的步骤

**第1步**

 	必须创建一个输入目录

```
$ $HADOOP_HOME/bin/hadoop fs -mkdir /user/input 
```

**第2步**

 	传输并使用本地系统put命令，Hadoop文件系统中存储的数据文件

```
$ $HADOOP_HOME/bin/hadoop fs -put /home/file.txt /user/input 
```

**第3步**

 	可以使用ls命令验证文件。

```
$ $HADOOP_HOME/bin/hadoop fs -ls /user/input 
```

##### 从HDFS中检索数据

 	假设在HDFS文件名为outfile。下面给出的是一个简单的演示用于检索从Hadoop文件系统所需的文件。

**第1步**

 	最初，使用cat命令来查看来自HDFS的数据。

```
$ $HADOOP_HOME/bin/hadoop fs -cat /user/output/outfile 
```

**第2步**

 	从HDFS得到文件使用get命令在本地文件系统。

```
$ $HADOOP_HOME/bin/hadoop fs -get /user/output/ /home/hadoop_tp/ 
```

##### 关闭HDFS

 	可以使用下面的命令关闭HDFS。

```
$ stop-dfs.sh 
```

#### HDFS优势

* 分布式文件系统是大数据时代解决大规模数据存储问题的有效解决方案，HDFS 开源实现了GFS，可以利用由廉价硬件构成的计算机集群实现海量数据的分布式 存储 
* HDFS具有兼容廉价的硬件设备、流数据读写、大数据集、简单的文件模型、强 大的跨平台兼容性等特点。但是，也要注意到，HDFS也有自身的局限性，比如 不适合低延迟数据访问、无法高效存储大量小文件和不支持多用户写入及任意修 改文件等 
* 块是HDFS核心的概念，一个大的文件会被拆分成很多个块。HDFS采用抽象的块概念，具有支持大规模文件储、简化系统设计、适合数据备份等优点 HDFS采用了主从（Master/Slave）结构模型，一个HDFS集群包括一个名称节点和若干个数据节点。名称节点负责管理分布式文件系统的命名空间；数据节点是分布式文件系统HDFS的工节点，负责数据的存储和读取 
* HDFS采用了冗余数据存储，增强了数据可靠性，加快了数据传输速度。HDFS还采用了相应的数据存放、数据读取和数据复制策略，来提升系统整体读写响应性能。HDFS把硬件出错看作一种常态，设计了错误恢复机制

#### HDFS不适用以下场景

- 低延时的数据访问 
          对延时要求在毫秒级别的应用，不适合采用HDFS。HDFS是为==高吞吐数据传输==设计的,因此可能牺牲延时HBase更适合低延时的数据访问。
- 大量小文件 
          文件的元数据（如目录结构，文件block的节点列表，block-node mapping）保存在NameNode的内存中， 整个文件系统的文件数量会受限于NameNode的内存大小，经验而言，一个文件/目录/文件块一般占有150字节的元数据内存空间。如果有100万个文件，每个文件占用1个文件块，则需要大约300M的内存。因此十亿级别的文件数量在现有商用机器上难以支持
- 多方读写，需要任意的文件修改 

​                  HDFS采用追加（append-only）的方式写入数据。不支持文件任意offset的修改。不支持多个写入器