---
layout: post
title: "angular for backend developer"
date: 2016-11-29 10:24:48 +0800
comments: true
categories: 
---
## 内容概要

- 意义
- 核心概念
- 实例

##意义

Angular作为前端的MVC框架，直接利用了服务器端开发的概念，即表现层和逻辑层分离，数据和逻辑通过控制器分发，并且通过request, session, global 等来传递数据和模型(model)。
angular也提供了控制器、业务服务功能，并且通过相关的指令（Directive）实现类似JSP内嵌Java代码的功能。而数据在视图和控制器之间的传递则通过scope完成。
最关键的是，由于数据和视图都在浏览器中，给数据和视图的实时双向绑定提供了基础，所以angular的变量改变可以直接体现到视图中，反之亦然。

##核心概念
<!-- more -->

**依赖注入**

跟Spring的依赖注入类似，Angular也可以实现依赖注入，而且更简单，只要页面中引入了所有的js文件(如：controllerA.js, serviceB.js)，Angular会根据名字（同spring 里的依据名称注入），直接注入同名的参数。比如：

~~~~
app.controller("controllerA", function($scope, serviceB){
   //your code here
});
~~~~

angular 会自动注入serviceB的实例。

**控制器 (Controller)**
控制器是实现视图和逻辑分离的关键一环。Angular通过

`
angular.module(xxx).controller(xxx)
`

来定义控制器。

**模块 (Module)**
Angular实现代码隔离的基本单位，所有的功能（包括控制器、service等）都必须在一个模块内定义。一个模块可以以来其他的模块，例如：
~~~~
var app = angular.module("moduleName",["depency1", "module2"]).
~~~~
通常一个SPA (Single Page Application)只用一个module就足够了。

**服务 (Service)**
对应到服务器端的Service层，主要是对逻辑的一种封装。Angular中通过
~~~~
angular.module("moduleName").factory("ServiceName", function(paramters){});
~~~~
来定义。

**指令(Directive)**
指令类似语法糖，可以在HTML（或者叫Angular模板）中定义数据展示的行为。指令通常用在HTML元素上（这也是比JSP好的地方，页面其实并不完全以来angular，如果把Angular移走，）。常见的有：

- ng-model: 绑定一个元素到一个变量，元素的值的改变会导致变量的改变，反之亦然。
- ng-repeat: 最常见的一个指令。通常页面最常用的用例就是，循环一个集合，把集合里的数据展示成一个列表，如果有必要可以进行分页。
- ng-if: 顾名思义，就是在if内指定的表达式为true时才将一个元素渲染到界面上。
- ng-show：类似ng-if，区别在与，本指令会在表达式为真时显示，为假时隐藏，而不是完全不存在元素。
- ng-click：指定一个元素的点击事件，目标通常是controller里定义的function。
- ...

**插件**
Angular可以和其他插件集成，比如最常见的一个是ui-router。虽然Angular自带有router相关的指令，但是de facto的router还是用ui-router。
插件表现的形式有许多种，比如直接提供接口，也可以以自定义指令的形式出现。

过滤器 (Filter)
Angular中的过滤器跟服务器端的过滤器不是一个概念，我认为更准确的叫法应该叫管道过滤器，因为过滤器其实是对数据（尤其是表达式中的值），进入到pipe中，然后进行处理后输出。比如如下的例子将一个长整型的时间戳输出成易读的形式：
~~~~
{{ time | date: "yyyy-MM-DD hh:mm:ss z"}}
~~~~

**包管理器**
Angular并不依赖特定的JS包管理器，我目前用bower, 常用的命令有:
~~~~
bower install package-name --save
bower uninstall package-name --save
~~~~
 等。

也可以用npm，命令也和bower区别不大。

##一点感触

之前一直没有尝试一个前端项目的主要是有个错误的判断：以为angular等项目依赖npm/bower打包工具，打包成一个特殊入口的app。而我一直不知道怎么样打包，也没见到有任何文章提到过，所以一直感觉无从下手。直到看到了一个项目的源代码后才发现，原来根本不需要特殊的入口，跟普通的web page和js的关系并没有本质的区别，也是一个index.html 引用一个js(通常是app.js)，由这个js定义angular模块、控制器。在使用时，只要确保页面中有引入app.js, controller.js等angular就可以自动完成装配。