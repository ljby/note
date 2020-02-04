#### [Arthas](https://alibaba.github.io/arthas/)

* 写一个死锁代码，通过jps找到对应的死锁进程，通过jstack+端口号看具体分析

* 利用arthas进行分析：

  1. java -jar arthas-boot.jar 选择对应的端口进入arthas界面

  2. thead查看应用程序中所有线程的情况

  3. bashboard 查看实时查看应用监控数据，及当前系统和jvm的信息

  4. thread ID 查看线程的状态信息

     ```java
     thread 1 | grep 'main(' 打印线程ID 1的栈，通常是main函数的线程
     ```

  5. watch命令去查看方法的参数、返回值和异常信息

     ```java
     watch 全限定类名 方法名 观察点 参数
     //参数：-x n，指定观察点的展开的层数，默认为1
     watch 全限类名 方法 returnObj   
     //查看某一方法的返回值
     ```

     [常用观察点](https://alibaba.github.io/arthas/advice-class.html)

  6. sc命令查看类的信息

  7. Jad反编译类源码：可以查看线上运行的代码是否和你预期的一样

  8. mc 内存编译，.java文件---->.class文件

  9. stack 查看某一方法的调用堆栈

  10. ```java
      cat ~/logs/arthas/arthas.log  查看arthas日志
      http://127.0.0.1:8563         web控制台
      java -jar arthas-boot.jar -h  查看帮助
      退出当前的连接，用quit或者exit命令
      完全退出arthas，执行shutdown命令
      ```

[更多用法](https://www.cnblogs.com/cjsblog/p/10741651.html)