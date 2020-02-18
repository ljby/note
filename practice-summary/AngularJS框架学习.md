#### 简介

​        AngularJS的主要特点是 mvc 数据双向绑定 分模块 依赖注入	

```javascript
mvc
m: $Scope 变量
V:视图
c:controller=function(){} 控制器 方法
```

#### 扩展

AngularJS：通过 **ng-directives** 扩展了 HTML。

**ng-app**： 指令定义一个 AngularJS 应用程序。

**ng-model**： 指令把元素值（比如输入域的值）绑定到应用程序。

**ng-bind**： 指令把应用程序数据绑定到 HTML 视图

**ng-init**=“变量名=‘变量值’” 一般加在body的起始标签内 初始化变量 也可以初始化方法 相当于onload方法

```javascript
//表达式：
AngularJS 表达式写在双大括号内：{{ expression }}

//字符串
div ng-app="" ng-init="firstName='John';lastName='Doe'">
<p>姓名： {{ firstName + " " + lastName }}</p>
</div>

//对象
<div ng-app="" ng-init="person={firstName:'John',lastName:'Doe'}">
<p>姓：{{ person.lastName }}</p>
</div>

//数组
<div ng-app="" ng-init="points=[1,15,19,2,40]">
<p>第三个值为 <span ng-bind="points[2]"></span></p>
</div>
```

AngularJS通过指令来扩展html，通过内置指令来添加功能，允许自定义指令

```javascript
<div ng-app="" ng-init="names=['Jani','Hege','Kai']">
  <p>使用 ng-repeat 来循环数组</p>
  <ul>
    <li ng-repeat="x in names">
      {{ x }}
    </li>
  </ul>
</div>
```

```javascript
//使用 .directive 函数来添加自定义的指令
<body ng-app="myApp">
<runoob-directive></runoob-directive>
/*或者类名<div class="runoob-directive"></div>
  属性：<div runoob-directive></div>
*/
<script>
var app = angular.module("myApp", []);
app.directive("runoobDirective", function() {
    return {
        template : "<h1>jby</h1>"
    };
});
</script>
</body>
```

限制指令只能通过特定的方式来调用，restrict : "A",默认通过元素名和属性名来调用指令

```javascript
//验证用户输入
<form ng-app="" name="myForm">
    Email:
    <input type="email" name="myAddress" ng-model="text">
    <span ng-show="myForm.myAddress.$error.email">不是一个合法的邮箱地址</span>
</form>
```

```javascript
//提供css类
<style>
input.ng-invalid {
    background-color: lightblue;
}
</style>
<body>
<form ng-app="" name="myForm">
    输入你的名字:
    <input name="myAddress" ng-model="text" required>
</form>
```

#### Scope

​        Scope(作用域) 是应用在 HTML (视图) 和 JavaScript (控制器)之间的纽带，Scope 是一个对象，有可用的方法和属性，Scope 可应用在视图和控制器上

