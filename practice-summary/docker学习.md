### docker背景

​         开发与运维部署之间的环境和配置问题，给出了一套标准的解决方案

​        "一次封装，到处运行"：解决了运行环境和配置问题软件容器，方便持续集成并有助于整体发布的容器虚拟化技术

### docker用途

​        虚拟机就是带环境安装的一种解决方案，可以在一种操作系统中运行另一种操作系统，缺点：资源占用多、冗余步骤多、启动慢

​        容器内的应用程序直接运行于宿主的内核，容器内没有自ide的内核，而且也没有进行硬件虚拟，所以容器的启动要比虚拟机快；每个容器之间相互隔离，每个容器都有自己的文件系统，容器之间进程不会互相影响，能区分计算资源

### docker的架构图

![2018030421154810](E:\java\Interview\shixi_by1\images\2018030421154810.jpg)

### dock的使用

* 安装docker 

* 去docker仓库找到这个软件对应的镜像

* 使用docker运行这个镜像，这个镜像就会生成一个docker容器

* 对容器的停止就是对软件的停止

### docker的三大要素

* 镜像：就是一个只读模板（类）
* 容器：是镜像的一个实例（对象）Container，每个容器都是相互隔离、保证安全的平台，它可以被启动、开始、停止、删除（简易版的linux环境）
* 仓库：集中存放镜像文件的场所，仓库注册服务器上往往存放着多个仓库，每个仓库又包含多个镜像，分为私有仓库和公有仓库

### docker安装过程

​       centos7下

* 清理旧版本：

  ```shell
  yum remove docker \
                    docker-client \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-engine
  ```

* 进行安装

```shell
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum-config-manager --enable docker-ce-nightly
yum-config-manager --enable docker-ce-test
yum-config-manager --disable docker-ce-nightly
yum install docker-ce docker-ce-cli containerd.io
//启动docker
systemctl start docker
docker run hello-world

默认安装路径： docker 的镜像与容器都存储在 /var/lib/docker 下面
```

##### 配置阿里云/网易云镜像加速器

##### 配置阿里云加速器

##### 启动docker后台

​        ==docker run hello-world==（镜像名），先去本机找此镜像，然后去阿里云拉

#### docker底层原理

​      c/s结构，docker守护进程运行在主机上，然后通过Socket连接从客户端访问，守护线程从客户端接受命令并管理运行在主机上的容器

##### docker与vm比较

### docker常用命令

##### 帮助命令

* docker version：

* docker info：

* docker --help

##### 镜像命令

* **docker images**：列出本地主机上的镜像

```
docker images -a：列出本地所有的镜像（含中间映射层）
docker images -q:只显示当前镜像的id
docker images -qa
docker images --digests：显示镜像的摘要信息
docker images --no-trunc：显示完整的镜像信息
```

* **docker search   某个镜像的名字**：

  docker search -s 30 tomcat ：点赞数超过30的tomcat

* **docker pull  某个镜像的名字：** 

  docker pull tomcat[:latest]   默认拉最新的版本

* **docker rmi   某个镜像名字id：**   删除某个镜像

  -f：强制删除

  docker rmi  -f  $(docker images -qa)：全部删除

* **docker commit**

  提交容器副本使之成为一个新的镜像

  docker commit -m "提交的描述信息"  -a=”作者“  容器ID要创建的目标镜像名:[标签名]

  ```shell
  docker exec  -it  容器ID  /bin/bash
  ls
  cd   webapps
  rm -rf docs     //删除了tomcat的document
  docker ps
  docker commit  -a=”jby“  -m=”tomcat without  docs“   容器ID   命名空间/tomcat：1.1
  
  #运行新的镜像
  docker run -it 容器名：版本
  ```

#####  容器命令

​    git pull centos   //拉centos

* 新建并启动容器
    **docker run -it  镜像ID**

    **docker run  [options]  images  [command]**

    options说明：
     --name：为容器指定一个名称
     -d:后台运行容器，并返回容器ID，即启动守护线程
     ====-i:以交互式运行容器，通常与-t同时使用==
     ==-t：为容器分配一个伪输入终端，通常与-i同时使用====
     -P:随机端口映射
     -p：指定端口映射   **docker run -it -p 8080 tomcat**

* docker ps [options]:列出所有在运行的容器信息
   options说明:
     -a :显示所有的容器，包括未运行的。
     -f :根据条件过滤显示的内容。
     --format :指定返回值的模板文件。
     -l :显示最近创建的容器。
     -n :列出最近创建的n个容器。
     --no-trunc :不截断输出。
     -q :静默模式，只显示容器编号。
     -s :显示总的文件大小。
* 退出容器
    exit -->关闭容器后退出
    ctrl+P+Q -->容器不关闭，退出
* 启动容器
    docker start 容器ID
* 重启容器
    docker restart 容器ID
* 停止容器
    docker stop 容器ID
* 强制停止容器
    docker kill 容器ID
* 删除已经停止的容器
    docker rm 容器ID
* 一次删除多个容器
    docker rm -f ${docker ps -a -q}
    docker ps -a -q | xargs docker rm

**启动式不交互：**

1.docker run -d 容器：

启动成功，但是docker ps未显示

2.查看容器日志

docker logs -f -t --tail 容器ID

3.查看容器内运行的进程

docker top 容器ID

4.查看容器内部细节

docker inspect 容器ID

5.进入正在运行的容器并以命令行交互

docker attach 容器ID

docker exec  -t 容器ID ls -l    （直接在宿主机上运行，不用进入到容器）

![1557160354750](C:\Users\y1291\AppData\Roaming\Typora\typora-user-images\1557160354750.png)

6.从容器内拷贝文件到主机上

   docker cp 容器ID：容器内路径  目的主机路径

### docker镜像

​        是一种轻量级的、可执行的独立软件包，用来打包软件环境和基于运行环境开发的软件，包含用于某个软件所需的所有内容

​        高性能的文件系统，支持对文件系统的修改作为一次提交来一层层的叠加，同时可将不同目录挂载到同一个虚拟文件系统下

![1557161033127](C:\Users\y1291\AppData\Roaming\Typora\typora-user-images\1557161033127.png)

docker镜像采用==分层==

![1557161567208](C:\Users\y1291\AppData\Roaming\Typora\typora-user-images\1557161567208.png)

镜像的特点

![1557161703837](C:\Users\y1291\AppData\Roaming\Typora\typora-user-images\1557161703837.png)

### 容器数据卷

​        就是将docker容器运行产生的数据持久化，完全独立于容器的生命周期

​         卷就是目录或文件，存在于一个或多个容器中，由docker挂载到容器中，但不属于联合文件系统

* 数据卷可在容器之间共享或重用数据

* 卷中的更改可以直接生效

* 数据卷中的更改不会包含在镜像的更新中

* 数据卷的生命周期一直持续到没有容器使用它为止

  ##### 直接命令添加

  1.命令：

  docker run -it  -v /宿主机绝对路径目录：/容器内目录   镜像名

  2.检验挂载是否成功：

  docker ps

  docker  inspect  容器ID        “Volumes卷”

  3.容器和宿主机之间数据共享

  4.容器停止退出后，主机修改后数据是否同步：同步

​        5.docker run -it  -v /宿主机绝对路径目录：/容器内目录：ro   镜像名

​            容器只能读不能写不能创建目录/文件

##### DockerFile解析

​        理解为对docker images源码级的描述

```shell
自己创建一个dockerfile:
#1.在根目录下新建mydocker文件夹进入
   mkdir /mydocker
   cd /mydocker
#2.可在dockerfile中使用volumes指令来给镜像添加一个或多个数据卷
#3.file构建
    vim dockerfile
    代码：
    # volume test
    from  centos
    volume ["/dataVolumeContainer1","/dataVolumeContainer2"]
    CMD echo "finished,-------success"
    CMD /bin/bash
    相当于=>{ docker run -it -v  /host:/dataVolumeContainer1 -v /host:/dataVolumeContainer1  contos /bin/bash }
#4.build后生成镜像，获得一个新镜像
     docker build -f /mydocker/dockerfile -t jby/centos .
#5.run容器
     docker run -it jby/centos    此时这个容器有两个数据卷
#6.查找容器内的卷目录地址对应的宿主机目录地址
    docker ps
    docker inspect 容器ID
#7.主机对应默认地址
```

#### 数据卷容器

​         命名的容器挂载数据卷，其他容器通过挂载这个（父容器）实现数据共享，挂载数据卷的容器，即为数据卷容器

```shell
docker run -it --name dec01  jby/centos
#02、03继承01
docker run -it --name dc02 --volumes-from dc01 jby/centos
docker run -it --name dc03 --volumes-from dc01 jby/centos
#删除掉01，03.02不受影响，数据共享是ok的，容器之间配置信息的传递，数据卷的生命周期一直持续到没有容器使用它为止
```

### DockerFile详解

​         dockerfile是用来构建docker镜像的构建文件，是由一系列命令和参数构成的脚本

​        构建三步骤：==编写dockerfile、docker build、docker run==

#### dockerfile构建过程解析

##### dockerfile内容基础知识

* 每条保留字指令都必须为大写字母且后面要跟随至少一个参数
* 指令按照从上到下，顺序执行
* #表示注解
* 每条指令都会创建一个新的镜像层，并对镜像进行提交

##### docker执行dockerfile的流程

* docker从基础镜像运行一个容器
* 执行一条指令并对容器进行修改
* 执行类似于docker commit的操作提交一个新的镜像层
* docker再基于刚提交的镜像运行一个新容器
* 执行dockerfile中的下一条指令直至所有指令都执行完成

#### Dockerfile的保留字指令

* FROM

  基础镜像，当前镜像是基于那个镜像的

* MAINTAINER

  镜像维护者的姓名和邮箱

* RUN

  容器构建时需要运行的命令

* EXPOSE

  当前容器对外暴漏的端口号

* WORKDIR

  指定在创建容器后，终端默认登录的进来工作目录，一个落脚点

* ENV

  用来在构建镜像的过程中设置环境变量

* ADD

  将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包

* COPY

  类似于ADD，拷贝文件和目录到镜像中，将从构建上下文目录中<源路径>的文件/目录复制到新的一层的镜像内的<目标路径>位置

* VOLUME

  容器数据卷，用于数据保存和持久化工作

* CMD

  指定一个容器启动时要运行的命令，可以有多个命令，但只有最后一个生效，CMD会被 docker run之后的参数替换

* ENTRYPOINT

  指定一个容器启动时要运行的命令，不会替换，而是追加

* ONBUILD

  当构建一个被继承的dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发

##### 案列

1.Base镜像：scratch，docker hub中99%的镜像都是通过在base镜像中安装和配置需要的软件构建出来的

2.改写Dockerfile，配置自己所需的环境

```shell
#编写dockerfile文件
cd /mycentos
vim Dickerfile
  #代码：
  FROM centos
  MAINTAINER jby<1047597200@qq.com>
  ENV  MYPATH /usr/local
  WORKDIR  $MYPATH
  RUN yum -y install vim
  RUN yum -y install net-tools
  EXPOSE 80
  CMD echo $MYPATH
  CMD echo "success------ok"
  CMD /bin/bash
#build
docker build -f /mycentos/Dickerfile -t  mycentos:1.3 .
#运行
docker images mycentos
docker run -it mycentos：1.3
```

3.列出镜像的变更历史

   docker  history 镜像ID  （从下往上看）

##### 自定义tomcat

 编写dockerfile文件

```shell
docker build -f dockerfile  -t  mytomcat:1.0

docker run -d -p 9080:8080 --name myt1 
 -v /mydockerfile/tomcat9/test:/usr/local/apache-tomcat-9.0.8/webapps/test
-v /mydockerfile/tomcat9/tomcat9logs:/usr/local/apache-tomcat-9.0.8/logs  --privileged=true 镜像名
```

将tomcat1.0测试的web服务test发布

```shell
cd /test
mkdir WEB-INF
cd WEB-INF/
vim web.xml
cd ..
vim a.jsp
docker restart 容器ID
localhost：9080/test/a.jsp
```

### 安装mysql、redis

* 搜索镜像

* 拉取镜像

* 查看镜像

* 启动镜像

* 停止容器

* 移除容器

```shell
docker pull mysql:5.6

docker run -p 12345：3306 --name mysql -v /dockersql/mysql/conf:/etc/mysql/conf.d     -v /dockersql/mysql/logs:/logs
-v /../mysql/data:/var/lib/mysql
-e MYSQL_ROOT_PASSWORD=123456   -d  mysql:5.6
```

```shell
docker pull redis:3.2

docker run -p 6379:6379 --name dockerredis -v /dockersql/myredis/data:/data     -v /dockersql/myredis/conf/redis.conf:/usr/local/etc/redis/redis.conf
-d redis:3.2 redis-server /usr/local/etc/redis/redis.conf
--appendonly yes

在主机/dockersql/myredis/conf/redis.conf目录下新建redis.conf文件
vim /dockersql/myredis/conf/redis.conf

docker exec -it 容器ID  redis-cli
```

#### 本机镜像发布到阿里云

##### 流程

```shell
docker commit -a jby -m "new mycentos with vim and ifconfig" 容器ID 名字 
#创建镜像仓库,登录
docker login --username= registry.cn-hangzhou.aliyuncs.com

docker tag  [imageID] registry.cn-hangzhou.aliyuncs.com/../mycentos:版本号

docker push registry.cn-hangzhou.aliyuncs.com/../mycentos:版本号
#将阿里云上的镜像拉到本地 
docker pull registry.cn-hangzhou.aliyuncs.com/../mycentos:版本号
```

