#### 使用Log4J目的

​        监视代码中变量的变化情况，周期性的记录到文件中供其他应用进行统计分析工作；跟踪代码运行时轨迹，作为日后审计的依据；担当集成开发环境中的调试器的作用，向文件或控制台打印代码的调试信息

​        Log4j有三个主要的组件：Loggers(记录器)，Appenders(输出源)和Layouts(布局)，这里可简单理解为日志类别，日志要输出的地方和日志以何种形式输出。综合使用这三个组件可以轻松的记录信息的类型和级别，并可以在运行时控制日志输出的样式和位置

#### Appender

Appender：日志输出器，配置日志的输出级别、输出位置等，包括以下几类：

​    ConsoleAppender: 日志输出到控制台；
​    FileAppender：输出到文件；
​    RollingFileAppender：输出到文件，文件达到一定阈值时，自动备份日志文件;
​    DailyRollingFileAppender：可定期备份日志文件，默认一天一个文件，也可设置为每分钟一个、每小时一个；
​    WriterAppender：可自定义日志输出位置

#### Layouts

　   有时用户希望根据自己的喜好格式化自己的日志输出。Log4j可以在Appenders的后面附加Layouts来完成这个功能。Layouts提供了 四种日志输出样式，如根据HTML样式、自由指定样式、包含日志级别与信息的样式和包含日志时间、线程、类别等信息的样式等等。

```java
   //其语法表示为：
　　org.apache.log4j.HTMLLayout（以HTML表格形式布局），
　　org.apache.log4j.PatternLayout（可以灵活地指定布局模式），
　　org.apache.log4j.SimpleLayout（包含日志信息的级别和信息字符串），
　　org.apache.log4j.TTCCLayout（包含日志产生的时间、线程、类别等等信息）
　　
　　//配置时使用方式为：
　　log4j.appender.appenderName.layout =fully.qualified.name.of.layout.class
　　log4j.appender.appenderName.layout.option1 = value1
　　…
　　log4j.appender.appenderName.layout.option = valueN
```

#### 日志级别

一般日志级别包括：ALL，DEBUG， INFO， WARN， ERROR，FATAL，OFF
Log4J推荐使用：DEBUG， INFO， WARN， ERROR

#### 输出格式

Log4J最常用的日志输出格式为：org.apache.log4j.PatternLayOut，其主要参数为：

```java
%n - 换行
%m - 日志内容
%p - 日志级别(FATAL， ERROR，WARN， INFO，DEBUG or custom)
%r - 程序启动到现在的毫秒数
%t - 当前线程名
%d - 日期和时间, 一般使用格式 %d{yyyy-MM-dd HH:mm:ss， SSS}
%l - 输出日志事件的发生位置， 同 %F%L%C%M
%F - java 源文件名
%L - java 源码行数
%C - java 类名，%C{1} 输出最后一个元素
%M - java 方法名
```

**代码示例**

```java
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration PUBLIC "-//log4j/log4j Configuration//EN" "log4j.dtd">
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">

<!-- 日志输出到控制台 -->
<appender name="console" class="org.apache.log4j.ConsoleAppender">
    <!-- 日志输出格式 -->
    <layout class="org.apache.log4j.PatternLayout">
        <param name="ConversionPattern" value="[%p][%d{yyyy-MM-dd HH:mm:ss SSS}][%c]-[%m]%n"/>
    </layout>

    <!--过滤器设置输出的级别-->
    <filter class="org.apache.log4j.varia.LevelRangeFilter">
        <!-- 设置日志输出的最小级别 -->
        <param name="levelMin" value="INFO"/>
        <!-- 设置日志输出的最大级别 -->
        <param name="levelMax" value="ERROR"/>
    </filter>
</appender>

<!-- 输出日志到文件 -->
<appender name="fileAppender" class="org.apache.log4j.FileAppender">
    <!-- 输出文件全路径名-->
    <param name="File" value="/data/applogs/own/fileAppender.log"/>
    <!--是否在已存在的文件追加写：默认时true，若为false则每次启动都会删除并重新新建文件-->
    <param name="Append" value="false"/>
    <param name="Threshold" value="INFO"/>
    <!--是否启用缓存，默认false-->
    <param name="BufferedIO" value="false"/>
    <!--缓存大小，依赖上一个参数(bufferedIO), 默认缓存大小8K  -->
    <param name="BufferSize" value="512"/>
    <!-- 日志输出格式 -->
    <layout class="org.apache.log4j.PatternLayout">
        <param name="ConversionPattern" value="[%p][%d{yyyy-MM-dd HH:mm:ss SSS}][%c]-[%m]%n"/>
    </layout>
</appender>

<!-- 输出日志到文件，当文件大小达到一定阈值时，自动备份 -->
<!-- FileAppender子类 -->
<appender name="rollingAppender" class="org.apache.log4j.RollingFileAppender">
    <!-- 日志文件全路径名 -->
    <param name="File" value="/data/applogs/RollingFileAppender.log" />
    <!--是否在已存在的文件追加写：默认时true，若为false则每次启动都会删除并重新新建文件-->
    <param name="Append" value="true" />
    <!-- 保存备份日志的最大个数，默认值是：1  -->
    <param name="MaxBackupIndex" value="10" />
    <!-- 设置当日志文件达到此阈值的时候自动回滚，单位可以是KB，MB，GB，默认单位是KB，默认值是：10MB -->
    <param name="MaxFileSize" value="10KB" />
    <!-- 设置日志输出的样式 -->`
    <layout class="org.apache.log4j.PatternLayout">
        <!-- 日志输出格式 -->
        <param name="ConversionPattern" value="[%d{yyyy-MM-dd HH:mm:ss:SSS}] [%-5p] [method:%l]%n%m%n%n" />
    </layout>
</appender>

<!-- 日志输出到文件，可以配置多久产生一个新的日志信息文件 -->
<appender name="dailyRollingAppender" class="org.apache.log4j.DailyRollingFileAppender">
    <!-- 文件文件全路径名 -->
    <param name="File" value="/data/applogs/own/dailyRollingAppender.log"/>
    <param name="Append" value="true" />
    <!-- 设置日志备份频率，默认：为每天一个日志文件 -->
    <param name="DatePattern" value="'.'yyyy-MM-dd'.log'" />

    <!--每分钟一个备份-->
    <!--<param name="DatePattern" value="'.'yyyy-MM-dd-HH-mm'.log'" />-->
    <layout class="org.apache.log4j.PatternLayout">
        <param name="ConversionPattern" value="[%p][%d{HH:mm:ss SSS}][%c]-[%m]%n"/>
    </layout>
</appender>

<!--1. 指定logger的设置，additivity是否遵循缺省的继承机制
    2. 当additivity="false"时，root中的配置就失灵了，不遵循缺省的继承机制
    3. 代码中使用Logger.getLogger("logTest")获得此输出器，且不会使用根输出器-->
<logger name="logTest" additivity="false">
    <level value ="INFO"/>
    <appender-ref ref="dailyRollingAppender"/>
</logger>

<!-- 根logger的设置，若代码中未找到指定的logger，则会根据继承机制，使用根logger-->
<root>
    <appender-ref ref="console"/>
    <appender-ref ref="fileAppender"/>
    <appender-ef ref="rollingAppender"/>
    <appender-ref ref="dailyRollingAppender"/>
</root>

</log4j:configuration>

```

```java
//测试代码
@Component
public class LogTest {
    Logger logger = Logger.getLogger("logTest1");
@PostConstruct
public void test(){
    for (int i=0; i<1000; i++) {
        logger.info(i + "----Log.Info----");
        logger.info(i + "----Log.Info----");
        logger.info(i + "----Log.Info----");
    }
  }
}
```
#### 实战

```java
import org.apache.log4j.BasicConfigurator;
import org.apache.log4j.LogManager;
import org.apache.log4j.Logger;

public class UseLog4j {
    //日志记录器
    private static Logger logger = LogManager.getLogger(UseLog4j.class);
    public static void main(String[]args){
        //自动快速地使用缺省Log4j环境
        BasicConfigurator.configure();
        //打印日志信息
        logger.info("hello log4j !");
    }
}
```

