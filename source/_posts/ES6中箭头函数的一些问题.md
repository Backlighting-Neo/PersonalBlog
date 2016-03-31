title: ES6中箭头函数的一些问题
tags: []
categories: []
date: 2016-02-02 11:52:00
---
> 箭头函数（Arrow Function）是ES6中一个新的特性，这是一种新的创建js中匿名函数的方式，在有些语言中，这被称为lambdas函数。虽然他省去了我们输入'function'这个字符的过程，但是他真的只是function的一个语法糖吗

<!--more-->

箭头函数（Arrow Function）是ES6中一个新的特性，这是一种新的创建js中匿名函数的方式，在有些语言中，这被称为lambdas函数。虽然他省去了我们输入'function'这个字符的过程，但是他真的只是function的一个语法糖吗


# 定义

他的形式大概就是下面的样子：

	([param] [, param]) => {
	   statements
	}

	param => expression

最初接触箭头函数的时候，我仅仅认为他是一个语法糖，直到有一天我在写React的生命周期函数的时候，使用了 ()=>{} 的形式的之后，怎么也拿不到 this.props 才开始了对箭头函数的深入了解

# 真的只是语法糖吗

箭头函数其实并不是ES5中function的语法糖，他会保留在定义箭头函数时的上下文，比如this的指向。

# 解决的问题

首先我们来看这样一段

	function foo() {
	   setTimeout(function() {
	      console.log("id:", this.id);
	   },100);
	}

	foo.call( { id : 1 } )

很显然，在100ms过后，console并不能输出我们所期待的正确的结果

然而当我们使用箭头函数来完成这个过程的时候

	function foo() {
	   setTimeout( () => {
	      console.log("id:", this.id);
	   },100);
	}

	foo.call( { id: 1 } );

是可以按照预期的行为进行的，这正式得益于箭头函数保留了定义时的this，如果不使用箭头函数，也许

# 行为处理

在我之前的概念之中， ()=>{} 应该和 function(){} 的形式是一样的，但后来发现貌似this没有发生变化，先来做一个最简单的验证，我们使用Babel对ES6的箭头函数进行编译，看看它是怎么处理的

## Babel的行为

我们对如下的这段ES6代码进行Babel编译

	var foo = function(){
	  var subFoo = ()=>{
	    this.parms = 1;
	  }
	}

得到了这样的ES5代码

	"use strict";

	var foo = function foo() {
	  var _this = this;

	  var subFoo = function subFoo() {
	    _this.parms = 1;
	  };
	};

可以看到，为了使 this 作用域没有发生变化，在箭头函数被编译成ES5传统函数的同时，内部对于 this 的引用被变成了 _this ，并且在函数外，需要保留 this 作用域的地方，对 this 进行了变量缓存。但在真正的ES6原生支持中，也是这样做的吗？

## ES6原生行为

