### Maven

==Maven 是一个项目管理工具，可以对 Java 项目进行构建、依赖管理==

#### **以下所有的命令都要在项目的根目录下进行。**

​        Maven提供了一套命令，我们可以在dos小黑窗中使用，当对Maven项目使用这些命令的时候我们应该切换到该项目的根目录下。
命令一：

##### **mvn clean**

​       这个命令可以清除我们的target文件夹（这个文件夹存放编译后的.class文件）

命令二：

##### **mvn compile**

​       和上面的命令相反，这个命令是编译一个项目的，前提是我们当前命令行位置为该项目的根目录下。

命令三：

##### **mvn test**

​       这个命令可以进行单元测试，测试test文件夹下的方法（test文件夹下的java文件格式名为：XxxTest.java）

命令四：

##### **mvn package**

​        将项目打包，如果是java项目就打包为.jar文件，如果是web项目及打包成.war文件。

命令五：

##### **mvn install**

​         将一个项目打包放在本地仓库中，以便多个项目使用

#### 依赖范围

在我们设置依赖的时候，会有一项Scope，里面有：
![这里写图片描述](http://img.blog.csdn.net/20171219215122713?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzkyNjY5MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
compile，provided，runtime，test，system五项。
![这里写图片描述](http://img.blog.csdn.net/20171219215242189?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzkyNjY5MTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
（图片来自网络）

​        Maven默认的是compile，即对于编译classpath，测试classpath，运行时classpath 都需要这个jar包。
​        **尤其值得注意的是provided，这个就像servlet-api那样，我们编译测试都需要这个jar包，但是当上传到服务器的时候就不再需要了（Tomcat的lib下有），如果这里我们默认compile，那么当程序在服务器上运行的时候将出现jar包的冲突！**

#### maven pom

​          所有 POM 文件都需要 project 元素和三个必需字段：groupId，artifactId，version

#### maven生命周期

| 验证 validate | 验证项目 | 验证项目是否正确且所有必须信息是可用的                   |
| ------------- | -------- | -------------------------------------------------------- |
| 编译 compile  | 执行编译 | 源代码编译在此阶段完成                                   |
| 测试 Test     | 测试     | 使用适当的单元测试框架（例如JUnit）运行测试。            |
| 包装 package  | 打包     | 创建JAR/WAR包如在 pom.xml 中定义提及的包                 |
| 检查 verify   | 检查     | 对集成测试的结果进行检查，以保证质量达标                 |
| 安装 install  | 安装     | 安装打包的项目到本地仓库，以供其他项目使用               |
| 部署 deploy   | 部署     | 拷贝最终的工程包到远程仓库中，以共享给其他开发人员和工程 |

标准生命周期：clean项目清理处理、dafault（build）项目部署、site（项目站点文档创建处理）

#### Maven创建java项目

```java
mvn archetype:generate -DgroupId=com.companyname.bank -DartifactId=consumerBanking -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

参数说明：

- **-DgourpId**: 组织名，公司网址的反写 + 项目名称
- **-DartifactId**: 项目名-模块名
- **-DarchetypeArtifactId**: 指定 ArchetypeId，maven-archetype-quickstart，创建一个简单的 Java 应用
- **-DinteractiveMode**: 是否使用交互模式

#### maven快照

​         如果 Maven 以前下载过指定的版本文件，比如说 data-service:1.0，Maven 将不会再从仓库下载新的可用的 1.0 文件。若要下载更新的代码，data-service 的版本需要升到1.1。

​         快照的情况下，每次 app-ui 团队构建他们的项目时，Maven 将自动获取最新的快照(data-service:1.0-SNAPSHOT)

 maven 命令中使用 -U 参数强制 maven 现在最新的快照构建。

```java
mvn clean package -U


    现在 app-web-ui 和 app-desktop-ui 项目的团队要求不管 bus-core-api 项目何时变化，他们的构建过程都应当可以启动。使用快照可以确保最新的 bus-core-api 项目被使用，但要达到上面的要求，我们还需要做一些额外的工作
    可以使用两种方式：
在 bus-core-api 项目的 pom 文件中添加一个 post-build 目标操作来启动 app-web-ui 和 app-desktop-ui 项目的构建。
使用持续集成（CI） 服务器，比如 Hudson，来自行管理构建自动化。
```

### Gradle

​        Gradle在设计的时候基本沿用了Maven的这套依赖管理体系。不过它在引用依赖时还是进行了一些改进。首先引用依赖方面变得非常简洁

```java
dependencies {
 compile'org.hibernate:hibernate-core:3.6.7.Final'
 testCompile ‘junit:junit:4.+'
}
```

#### 依赖管理系统

​        在Maven的管理体系中，用GroupID、ArtifactID和Version组成的Coordination唯一标识一个依赖项。任何基于Maven构建的项目自身也必须定义这三项属性，生成的包可以是Jar包，也可以是War包或Ear包

一个典型的引用如下：

```java
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-jpa</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-thymeleaf</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```
​        这里 GroupID类似于C#中的namespace或者Java中的package，而ArtifactID相当于Class，Version相当于不同版本，如果Version忽略掉，将选择最新的版本链接

​        同时，存储这些组件的仓库有远程仓库和本地仓库之分，远程仓库可以是使用世界公用的central仓库，也可以使用Apache Nexus自建的私有仓库；本地仓库则在本地计算机上。通过Maven安装目录下的settings.xml文件可以配置本地仓库的路径，以及采用的远程仓库地址。Gradle在设计时沿用了Maven这种依赖管理体系，同时也引入了改进，让依赖变得更加简洁：

```java
dependencies {
    // This dependency is exported to consumers, that is to say found on their compile classpath.
    api 'org.apache.commons:commons-math3:3.6.1'
// This dependency is used internally, and not exposed to consumers on their own compile classpath.
implementation 'com.google.guava:guava:23.0'
 
// Use JUnit test framework
testImplementation 'junit:junit:4.12'
 
compile 'org.hibernate:hibernate-core:3.6.7.Final'
testCompile ‘junit:junit:4.+'
```
}
           另外，Maven和Gradle对依赖项的审视也有所不同。在Maven中，一个依赖项有6种scope，分别是compile、provided、runtime、test、system、import。其中compile为默认。而gradle将其简化为4种，compile、runtime、testCompile、testRuntime。如上述代码“testCompile ‘junit:junit:4.+'”，在Gradle中支持动态的版本依赖，在版本号后面使用+号可以实现动态的版本管理。在解决依赖冲突方面Gradle的实现机制更加明确，两者都采用的是传递性依赖，而如果多个依赖项指向同一个依赖项的不同版本时可能会引起依赖冲突，Maven处理起来较为繁琐，而Gradle先天具有比较明确的策略

#### 多模块构建

​          在面向服务的架构中，通常将一个项目分解为多个模块。在Maven中需要定义parent POM(Project Object Model)作为一组module的通用配置模型，在POM文件中可以使用<modules>标签来定义一组子模块。parent POM中的build配置以及依赖配置会自动继承给子module

​          Gradle也支持多模块构建，在parent的build.gradle中可以使用allprojects和subprojects代码块分别定义应用于所有项目或子项目中的配置。对于子模块中的定义放置在settings.gradle文件中，每一个模块代表project的对象实例，在parent的build.gradle中通过allproject或subprojects对这些对象进行操作，相比Maven更显灵活

```java
allprojects {
 task nice << { task -> println "I'm $task.project.name" }
}

执行命令gradle -q nice会依次打印出各模块的项目名称。
```

#### 一致的项目结构

​        Maven指定了一套项目目录结构作为标准的java项目结构，Gradle也沿用了这一标准的目录结构。如果在Gradle项目中使用了Maven项目结构的话，在Gradle中无需进行多余的配置，只需在文件中包括apply plugin:'java'，系统会自动识别source、resource、test source、test resource等相应资源

​        同时，Gradle作为JVM上的构建工具，也支持Groovy、Scala等源代码的构建，同样功能Maven通过一些插件也能达到目的，但配置方面Gradle更灵活

#### 一致的构建模型

​         为了解决Ant中对项目构建缺乏标准化的问题，Maven设置了标准的项目周期，构建周期：验证、初始化、生成原始数据、处理原始数据、生成资源、处理资源、编译、处理类、生成测试原始数据、处理测试原始数据、生成测试资源、处理测试资源、测试编译、处理测试类、测试、预定义包、生成包文件、预集成测试、集成测试、后集成测试、核实、安装、部署。但这种构建周期也是Maven应用的劣势。因为Maven将项目的构建周期限制过严，无法在构建周期中添加新的阶段，只能将插件绑定到已有的阶段上。而Gradle在构建模型上非常灵活，可以创建一个task，并随时通过depends建立与已有task的依赖关系

#### 插件机制

​          两者都采用了插件机制，Maven是基于XML进行配置，而在Gradle中更加灵活