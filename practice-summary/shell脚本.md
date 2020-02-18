#### Shell概述

1.自动化批量系统初始化（update、软件安装、时区设置、安全策略...）

2.自动化批量软件部署程序

3.管理应用程序（KVM，集群管理扩容，MySQL）

4.日志分析处理程序（PV,UV,grep/awk）

5.自动化备份恢复程序（MySQL完全备份/增量+Crond）

6.自动化管理程序

7.自动化信息采集及监控程序（收集系统/应用状态信息，CPU、Mem、Disk、Net，TCP Status，Apache，MySQL）

​        **shell是核心程序之外的指令解析器，是一个程序，同时是一种命令语言和程序设计语言，类型：ash、bash、ksh、csh、tcsh** 

==echo $shell==：判断其类型

#### 存取权限和安全

​       rwxrwxrwx：前三位--拥有者，中间三位--所属组，后三位--其他人(r:可读，w:可写，x：可执行)

​       (第一位)“-”：代表普通文件，“d”：代表目录文件，“l”：连接文件，“b”：块文件，“c”：字符文件，“p”：命令管道文件，“s”：socket文件

**改变文件的权限命令：**

* chmod：****

  #### 对文件进行操作
  
  ##### 分隔符进行分割提取
  
  ```shell
  cat xx.xx | awk -F ',' '{print $1"\t"$2"\t"$5"\t"$6"\t"$7}'
  
  #q取出第一列、第二列、第六列：先对某一列排序，取出指定的某一列
  sort -t\t -k6 ip6data.csv | gawk -F' ' '{
     if ($5 == 1){
         print $1"\t"$2"\t"$5"\t"$6
     }
   }' ip6data.csv > t6.txt
  ```
  
  ##### 统计csv文件中某一列的每个数据的个数
  
  ```shell
   cat xx.csv |awk -F ' ' '{arr[$5]+=1} END{for (no in arr) {print no,arr[no]} }' 
   或者
   cat xx.csv |awk -F ',' '{print $5}'|sort|uniq -c
  ```
  