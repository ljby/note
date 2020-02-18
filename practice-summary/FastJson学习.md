#### json

​        json是一种与语言无关的数据交换的格式

​        使用ajax进行前后台数据交换

​        移动端与服务端的数据交换

​        使用Json的格式与解析方便的可以表示一个对象信息，json有两种格式：

* 对象格式：{"key1":obj,"key2":obj,"key3":obj...}

* 数组/集合格式：[obj,obj,obj...]

 ```java
例如：user对象用json数据格式表示
{"username":"zhangsan","age":28,"password":"123","addr":"北京"}
List<Product> 用json数据格式表示
[{"pid":"10","pname":"小米"},{},{}]
 ```

​         只要是对象就用{括起来}，只要是集合就用[]括起来

**注意：**

​         对象格式和数组格式可以互相嵌套，一个对象中的一个属性可以是一个集合或数组

​         json的key是字符串，json的value是Object

[JSON验证工具](<http://www.bejson.com/>)

#### FastJson

​        fastjson.jar是alibaba开发的一款专门用于Java开发的包，可以方便的实现json对象与JavaBean对象的转换，实现JavaBean对象与json字符串的转换，实现json对象与json字符串的转换。除了这个fastjson以外，还有Google开发的Gson包，其他形式的如net.sf.json包，都可以实现json的转换。方法名称不同而已，最后的实现结果都是一样的

```java
将json字符串转化为json对象
在net.sf.json中是这么做的
JSONObject obj = new JSONObject().fromObject(jsonStr);//将json字符串转换为json对象
 
在fastjson中是这么做的
JSONObject obj=JSON.parseObject(jsonStr);//将json字符串转换为json对象
```

[源码分析](<https://blog.csdn.net/srj1095530512/article/details/82529759>)

##### 通过FastJson对海量数据的Json文件，边读取边解析