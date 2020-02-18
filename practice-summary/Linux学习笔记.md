# Linux学习笔记

### 1.文件系统

​	文件系统通常会将这两部分数据分别存放在不同的区块，==权限与属性放置到 inode 中，至于实际数据则放置在 data block 区块之中。==另外，还有一个超级区块（superblock）会记录整个文件系统的整体信息，包括 inode 与 block 的总量、使用量、剩余量等。

​	每个 inode 与 block 都有编号，至于这三个数据的意义可以简略说明如下：

- . superblock ：记录此 filesystem 的整体信息，包括 inode/block 的总量、使用量、剩余量，以及文件系统的格式与相关信息等；
- inode：记录文件的属性，一个文件占用一个 inode，同时记录此文件的数据所在的 block 号码。
- block：实际记录文件的内容，若文件太大是，会占用多个block。

​	    当系统加载一个文件到内存后，如果该文件**没有被更动过**，则在内存区段的**文件数据会被设定为干净(clean)的**。但如果内存中的文件数据**被更改过**了(例如你用nano去编辑过这个文件)，此时该内存中的数据会被**设定为脏的(Dirty)**。 此时所有的动作都还在内存中执行，并没有写入到磁盘中!系统会**不定时的将内存中设定为[DityJ 的数据写回磁盘**，**以保持磁盘与内存数据的-致性**。你也 可以利用第四章谈到的==sync指令来手动强迫写入磁盘==。

​		我们知道内存的速度要比磁盘快的多，因此如果能够将常用的文件放置到内存当中，这不就会增加系统性能吗?没错! 是有这样的想法!因此我们Linux 系统上面文件系统与内存有非常大的关系喔:

      ● 系统会将常用的文件数据放置到主存储器的缓冲区，以加速文件系统的读/写;因此Linux 的物理内存最后都会被用光!这是正常的情况!可加速系统效能;
      
      ● 你可以手动使用sync 来强迫内存中设定为Dirty 的文件回写到磁盘中; 
      
      ● 若正常关机时，关机指令会主动呼叫sync 来将内存的数据回写入磁盘内;
    
      ● 但若不正常关机(如跳电、当机或其他不明原因)，由于数据尚未回写到磁盘内，因此重新启动后可能会花
    
      ● 很多时间在进行磁盘检验，甚至可能导致文件系统的损毁(非磁盘损毁)。
### 2.RPM指令：

```linux
rpm -qa | grep X 	查询rpm软件包
rpm -q 软件包名 	查询软件包是否安装
rpm -qi 软件包名 查询软件包信息

rpm -ql 软件包名 查询软件包中的文件
rpm -ql firefox

rpm -qf 文件全路径名称 ： 查询文件所属的软件包

卸载 rpm 包
rpm -e 名称
```

### 3.yum指令：（必须联网）

```linux
查看 yum 有没有某个rpm包
yum list | grep firefox 
安装软件
yum install XXXX
```

### 4.压缩指令：（gzip，zip， tar）

```linux
gzip 压缩文件（压缩完成后不会保留原文件） gunzip 解压缩

zip -r 名称 要压缩的目录或文件 
zip -r mypackage.zip /home/ 
unzip 解压缩
unzip -d /opt/tmp mypackage.zip 将mypackage.zip解压缩到/opt/tmp下

tar 指令：
-c 	产生.tar打包文件
-v	显示详细信息	
-f	指定压缩后的文件名
-z	打包同时压缩
-x	解包.tar文件
-j 	针对*.tar.bz2

案例一：压缩文件
tar -zcvf a.tar.gz(压缩后的名称是什么) a1.txt a2.txt(那些文件要被压缩)
案例二：将 /home 的文件夹压缩成 myhome.tar.gz
tar -zcvf myhome.tar.gz /home/
案例三：将 a.tar.gz 解压到当前目录
tar -zxvf a.tar.gz
案例四：将 a.tar.gz 解压到/opt/tmp目录下
tar -zxvf a.tar.gz -C /opt/tmp/  保证该目录必须存在

tar 解压 bz2 类型的文件 
tar -jxvf filename

```

==tar指令：在解压到指定目录时保证该目录必须存在==

## ==Shell脚本(important)==

### 第一个shell程序：

```shell
#!/bin/bash
echo "hello world!"
echo "PATH=$PATH"
echo "user=$USER"
```

### shell 的变量：

#### 系统变量：

​	$HOME, $PWD, $SHELL, $USER 等等

``` shell
echo $HOME 
set 显示当前 shell 中的所有变量
```

#### 自定义变量：

##### 基本语法：

1. 定义变量：变量=值
2. 撤销变量： unset 变量
3. ==声明静态变量： readonly 变量，注意：不能 unset==
4. 多行注释

```shell
多行注释：
:<<!  
内容内容 内容内容 内容内容 内容内容  
!
```

##### 快速入门：

​	案例1：定义变量A 撤销变量A

```shell
#!/bin/bash
A=100 (自定义变量)
echo "A=$A"
unset A
echo "A=$A"
输出结果为：
A=100
A=
```

​	案例2：声明静态的变量 B=2， 不能 unset

```shell
#!/bin/bash
readonly B=99
echo "B=$B"
unset B
echo "B=$B"

输出结果为：
B=99
./myshell1.sh: 第 13 行:unset: B: 无法反设定: 只读 variable
B=99
（输入 set nu 即可查看行号）

```

​	案例3：可把变量提升为全局变量，可供其他 shell 程序使用

##### 变量命名规则：

```java
1. 变量名称可以由字母、数字、下划线组成、但是不能以数字开头
2. 等号两侧不能有空格
3. 变量名称一般习惯为大写
```

##### 将命令的返回值赋给变量（important）

```shell
1. A=`ls -al` 反引号，运行里面的命令，并把结果返回给变量A
2. a=$（ls -al） 等价于反引号
例：
RESULT1=`ls -al /home`
echo $RESULT1
echo "======QAQ===============QAQ====================QAQ============"
RESULT2=$(ls -al /home)
echo $RESULT2
```

##### 设置环境变量

```shell
1. export 变量名=变量值 （功能描述：将shell变量输出为环境变量）
2. source 配置文件	（让修改后的配置信息立即生效）
3. echo $变量名	（查询环境变量的值）

案例：
1. 在 /etc/profile 文件中定义TOMCAT_HOME环境变量
2. 查看环境变量TOMCAT_HOME的值
3. 在另外一个 shell 程序中使用 TOMCAT_HOME
4. source /etc/profile (为了让刚才的配置立即生效) 刷新的意思

TOMCAT_HOME=/opt/tomcat
export TOMCAT_HOME

最后在 source /etc/profile
```

##### 位置参数变量

​	当我们执行一个 shell 脚本时，如果希望获取到命令行的参数信息，就可以使用到位置参数变量。

比如： ./myshell1.sh 100 200。这就是一个执行 shell 的命令行，可以在 myshell 脚本中获取到参数信息。  

```shell
基本语法：
$n:
(n 为数字， $0 代表命令本身，$1-$9代表第一到第九个参数， 十以上的参数都需要用大括号包含，如${10} )

$* (这个变量代表命令行中的所有参数，$* 把所有的参数看成一个整体)

$@ (这个变量代表命令行中的所有参数，$@ 把所有的参数区分对待)

$# (代表所有参数的个数)
```

案例：编写一个 shell 脚本 position.sh,测试以上几个语法

```shell
#!/bin/bash
#获取到各个参数
echo "$0 $1 $2"
echo "$*"
echo "$@"
echo "参数个数是： $#"

执行：./ position.sh 30 60
输出：
./ position.sh 30 60
30 60
30 60 
参数个数是： 2
```

##### 预定义变量

就是shell 设计者实现已经定义好的变量，可以直接在 shell 脚本中使用

```shell
基本语法：
$$	当前进程号的PID
$!	后台运行的最后一个进程的PID
$?	最后一次执行命令的返回状态，如果这个值是0，证明上一个命令正确执行；如果这个变量的值为非0，则证明不正确执行。
```

案例：

```shell
vim preVar.sh
echo "当前的进程号 = $$"
#后台的方式运行 myshell1.sh 这个脚本
./myshell1.sh &&
echo "后台的最后一个进程号为： $!"
echo "执行的值= $?"
```

### 运算符

```shell
案例一：普通运算
#!/bin/bash
#运算符

#第一种方式 $()
RESULT1=$(((2+3)*4))
echo "$RESULT1"

#第二种方式 $[]
RESULT2=$[(2+3)*4]
echo "$RESULT2"

#第三种方式  expr
TEMP=`expr 2 + 3`
RESULT3=`expr $TEMP \* 4`
echo "$RESULT3"
案例二：求出两个参数的和
RESULT4=$[$1+$2]
echo "参数的和为： $RESULT4"
```

### ||、 &&、[ ]

```shell
用法：
&&运算符:
command1  && command2
&&左边的命令（命令1）返回真(即返回0，成功被执行）后，&&右边的命令（命令2）才能够被执行；
换句话说，“如果这个命令执行成功&&那么执行这个命令”。

语法格式如下：
 
    command1 && command2 [&& command3 ...]
 
1 命令之间使用 && 连接，实现逻辑与的功能。
2 只有在 && 左边的命令返回真（命令返回值 $? == 0），&& 右边的命令才会被执行。
3 只要有一个命令返回假（命令返回值 $? == 1），后面的命令就不会被执行。

||运算符:

command1 || command2
 
||则与&&相反。如果||左边的命令（命令1）未执行成功，那么就执行||右边的命令（命令2）；
或者换句话说，“如果这个命令执行失败了||那么就执行这个命令。

1 命令之间使用 || 连接，实现逻辑或的功能。
2 只有在 || 左边的命令返回假（命令返回值 $? == 1），|| 右边的命令才会被执行。这和 c 语言中的逻辑或语法功能相同，即实现短路逻辑或操作。
3 只要有一个命令返回真（命令返回值 $? == 0），后面的命令就不会被执行。

[ ] 补充
[ $var1 -ne 0 -a $var2 -gt 2 ]  # 使用逻辑与 -a
[ $var1 -ne 0 -o $var2 -gt 2 ]  # 使用逻辑或 -o
```



### 条件判断

```shell
基本语法
[空格condition空格] 
#非空返回true，可使用$?进行验证（0 为 true， > 1 为false）

案例：
[ Tyrone ] 返回 ture
[] 返回false
[ condition ] && echo "OK" || echo "not OK" 	条件满足，执行后面的语句
```

```shell
判断语句
1) 两个整数的比较
== 字符串比较
-lt 小于
-le 小于等于
-eq 等于
-gt 大于
-ge 大于等于
-ne 不等于
2） 按照文件权限进行判断
-r 有读的权限
-w 有写的权限
-x 有执行的权限
3）按照文件类型进行判断
-f 文件存在并且是一个常规的文件
-e 文件存在
-d 文件存在并是一个目录

应用实例：
案例1："ok" 是否等于 "ok"
案例2：23 是否大于等于 22
案例3：/shell/zy.txt 文件是否存在

#!/bin/bash

#案例1：
if [ "ok" = "ok" ]
then
        echo "equal"
fi

#案例2：
if [ 23 -gt 22 ]
then
        echo "大于等于"
fi
#案例3
if [ -e /shell/zy.txt ]
then
        echo "存在"
fi
```

### 流程控制

#### if 判断：

```shell
基本语法1：
if [ 条件判断式 ] ；then
	程序
fi

基本语法2：
if [ 条件判断式 ]
then
	程序
elif [ 条件判断式 ]
then
	程序
fi

案例 判断输入的数字是不是 大于等于60：

#!/bin/bash

if [ $1 -ge 60 ]
then
        echo "大于等于60"
elif [ $1 -lt 60 ]
then
        echo "不大于60"
fi

```

```shell
多重if ：
if [ condition1 ]; then
	程序
elif [ condition2 ]; then
	程序
elif [ condition3 ]; then
	程序
......
else
	程序
fi

案例：
#!/bin/bash
read -p "Input: " n

if [ $n -gt 50 ]; then
        echo ">50"
elif [ $n -gt 40 ]; then
        echo ">40"
else
        echo "<=40"
fi

```

#### case 判断

```shell
#!/bin/bash
#案例1：参数是1时，输出周一，2输出周二，其他输出“other”

case $1 in
"1")
        echo "周一"
;;
"2")
        echo "周二"
;;
*)
        echo "other"
;;
esac
```

#### for循环

```shell
基本语法 1 ：
for 变量 in 值1 值2 值3...
	do
		程序
	done
#案例
打印命令行输入的参数	[会使用到 $* $@]

#!/bin/bash

#使用 $*
for i in "$*"
do
        echo "the num is $i"
done

#使用 $@
for i in "$@"
do
        echo "the num is $i"
done
```

```shell
基本语法2：
for （（ 初始值；循环控制条件；变量变化））
do
	程序
done

案例：从 1 加到 100
#!/bin/bash
#1 加到 100
SUM=0
for ((i=1;i<=100;i++))
do
        SUM=$[$SUM+$i]
done
echo "sum=$SUM"
```

#### while循环

```shell
基本语法 1 
while [ 条件判断式 ]
do
	程序
done

案例：输入一个 n， 求出1 + 。。。。+ n的值

#!/bin/bash
SUM=0
i=0
while [ $i -le $1 ]
do
        SUM=$[$SUM+$i]
        i=$[$i+1]
done
echo "sum= $SUM"
```

### read读取控制台输入

```shell
基本语法：
read（选项）（参数）
选项：
-p:指定读取值时的提示符
-t:指定读取值时等待的时间（秒），如果没有在指定的时间内输入，就不再等待了
参数
变量：指定读取值的变量名

案例：
1. 读取控制台输入一个num值
2. 读取控制台输入一个num值，在10秒内输入

#!/bin/bash
read -p "Input:" NUM1
echo "输入的是：$NUM1"

read -t 4 -p "Input2:" NUM2
echo "输入的是：$NUM2"

```

### 函数：

#### basename

```shell
基本语法：
功能：返回完整路径最后/的部分，常用于获取文件名
basename [pathname][suffix]
basename [string][suffix]

选项：
suffix 为后缀，如果suffix被指定了，basename会将pathname或者string中的suffix去掉

应用实例：
案例1：
请返回 /home/aaa/test.txt 的 “test.txt” 部分

basename /home/aaa/test.txt 	输出test.txt

加上文件后缀 可以只获取文件名称不含后缀
basename /home/aas/test.txt txt 	输出 test
```

#### dirname

```shell
获取目录
dirname /home/aaa/test.txt	输出/home/aaa
```

#### 自定义函数：

```shell
基本语法：
[function] funname[()]
{
	Action:
	[return int;]
}
调用直接写函数名： funname[值]

案例：
计算输入两个参数的和(read)  getsum

#!/bin/bash
function getSum(){
        SUM=$[$n1+$n2]
        echo "SUM=$SUM"
}
read -p "Input1:" n1
read -p "Input2:" n2

#调用getSum
getSum $n1 $n2
```

### sed

​        sed是一个“非交互式的”面向字符流的编辑器。能同时处理多个文件多行的内容，可以不对原文件改动，把整个文件输入到屏幕,可以把只匹配到模式的内容输入到屏幕上。还可以对原文件改动，但是不会再屏幕上返回结果。

​        sed命令告诉sed对指定行进行何种操作,包括打印，删除，修改等。

**sed命令的语法格式：**

sed的命令格式： sed [option] 'sed command'filename

sed的脚本格式：sed [option] -f 'sed script'filename

**sed命令的选项(option)：**

选项：

a		在当前行后添加一行或者多行

c		用新文本修改（替换）当前行中的文本

d		删除行

i		在当前行之前插入文本

S		用一个字符串替换另一个

​		s	替换标志

​		g	全局替换

​		i 忽略大小写

r		从文件中读取

w		将行写入文件

<hr/>
l		列出非打印字符

p		打印行

n		读入下一输入行，并且从下一条命令而不是第一条命令开始对其的处理

q		结束或退出sed

！		对所选行以外的所有行应用命令

```shell
删除命令： d
sed -r '3d' datafile		删除第三行
sed -r '3{d;}' datafile 	删除第三行
sed -r '3{d}' datafile 		删除第三行
sed -r '/north/d' datafile	删除包含north的行
sed -r '/sout/d' datafile	删除包含sout的行

```

```shell
替换命令; s
sed -r 's/(.)(.*)/\1YYY\2/' datafile  在第一个字符和第二个字符之间加上YYY
sed -r 's/west/north/g' datafile
sed -r 's#west#north#g' datafile 把 west 替换成 north
sed -r 's/[0-9][0-9]$/&.5' datafile 把以两个数字结尾的后面加上 .5
```

```shell
修改命令：c a i 
sed -r '2c\ZYZYYZYZYYZ' datafile 把第二行替换成ZYZYZYYZYZ
sed -r '2i\ZYZYYZYZYYZ' datafile 在第二行前面加一行ZYZYZYYZYZ
sed -r '2a\ZYZYYZYZYYZ' datafile 在第二行后面加一行ZYZYZYYZYZ
```

```shell
修改命令 n
sed -r '/adm/{n;d}/' datafile 删除adm的下一行
sed -r '/adm/{n;s/.*/ZYZYYZYZYZY/}' datafile 把adm下一行的所有内容替换为 ZYZYZYZYYZ
```

```shell
sed 实战训练：
1. 删除文件中注释行
sed -ri '/^[ \t]*#/d' sed.txt  注释前面可能有空格 或者table键
2.删除文件中注释的 // 开始的行
sed -ri '\#^[ \t]*//#d' sed.txt
3. 删除空行
sed -ri '/^[ \t]*$/d' sed.txt
4. 删除注释行以及空行
sed -ri '/^[ \t]*($|#)/d' sed.txt
5. 给1,5行加上注释
sed -r '1,5s/^[ \t]*#*/#/' new.txt
6. sed 使用外部变量
sed -ri "3a$test" new.txt  单引号表示强引用 双引号弱点
单引号是纯字符
双引号会识别变量
sed -ri '$a'"$test" new.txt 在new.txt最后一行加上 test 这个变量

```

### awk

```shell
选项：
-F 定义输入字段分隔符，默认的分隔符是空格或者制表符
= = command
BEGIN{}		{}		END{}
行处理前	行处理		行处理后

#awk 'BEGIN{print 1/2} {print "OK"} END{print"end"}' new.txt	
输出：
0.5
ok
ok
ok
ok
。。。(ok 个数取决于文件的行数)
end

BEGIN{} 通常是用于定义一些变量，例如BEGIN{FS=":";OFS="---"}
FS 定义分隔符 OFS 定义输出间隔
案例：
awk '/root/' zy.txt 查找出包含 root的行 等于 grep 'root' zy.txt
awk -F: '{print $1}' zy.txt 用:分割 打印出第一个值
awk -F: '/root/{print$1,$3}' zy.txt 用:分割打印出 包含 root 的行的 第一个 和 第三个值

df -P | grep '/' | awk '$4>3500000 {print $4}'
```

### seq

**作用：**

​        seq命令用于以指定增量从首数开始打印数字到尾数，即产生从某个数到另外一个数之间的所有整数，并且可以对整数的格式、宽度、分割符号进行控制

**语法：**

　　[1] seq [选项]    尾数

　　[2] seq [选项]    首数  尾数

　　[3] seq [选项]    首数  增量 尾数

**选项：**

​    -f, --format=格式

​    -s, --separator=字符串，使用指定的字符串分割数字（默认使用个"\n"分割）

​    -w, --sequal-width  在列前添加0 使得宽度相同























