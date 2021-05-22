---
title: JS的词法作用域与this的问题
date: 2021-02-10 01:22:04
tags: 前端，Javascript
---
面试的时候遇到一道题
```
    inner = 'window';
    function say() {
        console.log(inner);
        console.log(this.inner);
    }

    var obj1 = (function() {
        var inner = '1-1';
        return {
            inner: '1-2',
            say: function() {
                console.log(inner);
                console.log(this.inner);
            }
        }
    })();

    var obj2 = (function() {
        var inner = '2-1';
        return {
            inner: '2-2',
            say: function() {
                console.log(inner);
                console.log(this.inner);
            }
        }
    })();
    say();
    obj1.say();
    obj2.say();
    obj1.say = say;
    obj1.say();
    obj1.say = obj2.say;
    obj1.say();
    // 程序输出？

```
这题考察的是词法作用域、闭包和this的指向。

首先需要了解词法作用域是由你在写代码时将变量和块作用域写在哪里来决定的，当词法分析器处理代码时会保持作用域不变（除了eval，with）
而this的指向一般都是在函数运行时绑定的（谁调用函数，this就指向谁）。下面分别分析每个`say()`的输出。

第一个`say()`输出`window window`。
因为在函数作用域内找不到inner，只能在全局作用域中找到。所以`inner = window`, 而`say()`调用的时候，是不带任何修饰的，所以应用了this的默认绑定，即this指向了全局对象window。值得一提的是，如果函数内使用了严格模式，this会绑定`undefined`。
```
var inner = 'window';
function say(){
    'use strict'
     console.log(this.inner);
}
say() // TypeError: Cannot read property 'inner' of undefined
```

第二个`obj1.say()`输出`1-1 1-2`。
首先`obj1.say()`是一个闭包，指向立即执行函数（IIFE）的作用域，所以在作用域内申明的`inner`没有因函数执行完毕而被垃圾回收。第一个`console.log`输出的是IIFE内部的inner。而`obj1.say()`在调用时对this进行了隐性绑定，即this绑定到执行上下文对象（这里为`obj1`），所以`this.inner = obj1.inner`。

第三个`obj2.say()`与第二个同理。输出`2-1 2-2`。

第四个`obj1.say()`输出`window 1-2`。
在调用之前，`obj1.say`指向了全局作用域中的`say`,此时不存在闭包，所以在函数`say`的词法作用域下，`inner = window`。由于执行上下文没变，依旧是`obj1`， `this.inner = obj1.inner`。

第五个`obj1.say()`输出`2-1 1-2`。
在调用之前，`obj1.say`指向了`obj2.say`,此时闭包所指向的作用域是`obj2`的IIFE，所以`inner = '2-1'`。`this.inner`依旧是`obj1.inner`。
