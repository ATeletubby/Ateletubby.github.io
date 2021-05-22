---
title: 尝试手写一个简易的vue框架
date: 2020-8-20 02:05:25
tags: 前端
---

Vue是一款帮助前端快读构建UI的渐进式框架。了解原理有助于项目的构建。


### 准备工作
创建了一个id为app的div节点，作为挂载点。子节点包含了\{{\}}，v-html，v-model，@click这几种绑定数据与事件的属性。
```
<div id="app">
    <div>{{ name }}</div>
    <div v-html="value"></div>
    <input v-model="value"/>
    <button @click="changeValue">changeValueTo1</button>
</div>
```
假设存在一个Vue类，用一个对象options作为其构建函数的参数，其中包括了el，data，methods三个属性。最后创建了一个Vue实例。
```
<script>
  class Vue {
    constructor(options){
        ...
    }
    ...
  }
  
  let options = {
    el: "#app",
    data:{
      name: 'Jerry',
      value: 100,
    },
    methods: {
      changeValue: function(){
        this.value = 1;
      }
    }
  }
  let app = new Vue(options);
</script>
```
之后要做的便是实现这个Vue类，能够让options.data和前面的视图层绑定在一起。

### 构造函数
定义$el,$options用来存放挂载点和vue实例数据。
$watchEvent和三个方法的作用在后文会解释。
```
constructor(options){
    this.$el = document.querySelector(options.el);
    this.$options = options;
    // 存放被劫持的节点
    this.$watchEvent = {}
    // 代理options.data中的数据
    this.proxyData();
    // 劫持事件
    this.observe();
    // 编译 
    this.compile(this.$el);
}
```

### 数据代理
将options.data里所有参数赋给vue实例，通过`Object.defineProperty`可以的getter和setter函数实现代理。这样可以通过实例直接访问或修改options中的数据。
```
proxyData(){
    for (let key in this.$options.data){
      Object.defineProperty(this,key, {
        configurable: false,
        enumerable: true,
        // value: '',
        // writable: true,     
        get(){
          return this.$options.data[key];
        },
        set(val){
          this.$options.data[key] = val;
        }
      })
    }
}
```
关于`Object.defineProperty`的参数和用法可以参考[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

### 数据侦听
下一步是对`$options.data`里的各个属性进行侦听。Vue2也是运用了`Object.defineProperty`来实现观察者模式。
```
observe(){
    let that = this;
    for (let key in this.$options.data){
      let value = this.$options.data[key];
      Object.defineProperty(this.$options.data, key, {
        configurable: false,
        enumerable: true,
        get(){
          console.log('获取数据');
          return value;
        },
        set(val){
          console.log('设置数据');
          value = val;
          if (that.$watchEvent[key]) {
            that.$watchEvent[key].forEach((item,index)=>{
              item.update();
            })
          }
        }
      })
    }
}
```
现在option.data中的属性都被监听着。获取和设置这些属性能够触发一些行为。


### 依赖收集
所有数据被监听后，还需要找到视图中与数据绑定的节点。

先创建一个Watch类，来监听与data绑定的dom节点。
```
class Watch{
  constructor(vm,key,node,attr,nodeType){
    this.vm = vm;
    // key 为绑定的vm触发的属性
    this.key = key;
    // vm[key]数据绑定的HTML节点
    this.node = node;
    // vm[key]绑定的html节点的属性名称
    this.attr = attr;
    // 节点类型
    this.nodeType = nodeType;
  }
  update(){
    console.log('更新视图');
    this.node[this.attr] = this.vm[this.key]; 
  }
}
```
在`constructor`中声明变量`$watchEvent`，用来存储数据与dom节点的联系。其数据结构如下所示
```
$watchEvent = {
    name: [Watch],
    value: [Watch,Watch,Watch]
}

```
可以看到，`$watchEvent`将data中的属性名作为key，watch实例数组作为value。其中每一个watch实例保存着依赖该数据的dom节点。

当一个数据变化后，该数据的setter被触发，然后setter再触发每一个Watch实例的update方法。
```
// observe()
set(val){
  console.log('触发设置内容事件');
  value = val;
  if (that.$watchEvent[key]) {
    that.$watchEvent[key].forEach((item,index)=>{
      item.update();
    })
  }
  // 触发更新事件
}
```
这样视图也会更新。

### 编译
建立好`$watchEvent`和`Watch`类后，需要将el节点里所有childNodes进行遍历，找到使用模板语法、v-html等自定义属性的Dom节点。通过nodeType来判断节点的类型。
```
Rnode.childNodes.forEach((node, index) => {
  if (node.nodeType == 1){
    //处理元素类型的节点
    ...
  } else if (node.nodeType == 3){
    //处理文本类型的节点
}
}
```
##### 元素类型的节点

对于元素类型的节点，主要是找到节点的自定义属性。如v-html，v-model。

如果节点使用了v-html，Watch将绑定的数据与节点的innerHTML联系起来。一旦数据变化，Watch触发update，节点的innerHTML也随之变化。
```
if (node.hasAttribute('v-html')){
  let vmKey = node.getAttribute('v-html').trim();
  node.innerHTML = this[vmKey];
  let watcher = new Watch(this, vmKey, node, 'innerHTML');
  if (this.$watchEvent[vmKey]){
    this.$watchEvent[vmKey].push(watcher);
  } else {
    this.$watchEvent[vmKey] = [];
    this.$watchEvent[vmKey].push(watcher);
  }
  // 删除节点事件
  node.removeAttribute('v-html');
}
```
对于使用v-model的节点，Watch将绑定的数据与节点的value联系起来，数据变化，value也变化。同时，对节点绑定监听事件，节点的value变化后，实例数据也跟着变化。这样便实现了双向绑定。
```
if (node.hasAttribute('v-model')){
  let vmKey = node.getAttribute('v-model').trim();
  node.value = this[vmKey];
  let watcher = new Watch(this, vmKey, node, 'value');
  if (this.$watchEvent[vmKey]){
    this.$watchEvent[vmKey].push(watcher);
  } else {
    this.$watchEvent[vmKey] = [];
    this.$watchEvent[vmKey].push(watcher);
  }
  // 绑定input监听事件
  node.addEventListener('input', (event)=>{
    this[vmKey] = node.value;
  })
  node.removeAttribute('v-model');
}
```

如果遇到事件属性，为该节点绑定该事件。

```
if (node.hasAttribute('@click')){
  let vmKey = node.getAttribute('@click').trim();
  node.addEventListener('click',(event)=>{
    this.eventFn = this.$options.methods[vmKey].bind(this)();
  })
}
```
如果元素类型的节点存在子节点，需要对该节点递归调用compile方法。
```
// 元素类型的节点还有子节点
if (node.childNodes.length > 0){
  this.compile(node);
}
```
##### 文本类型的节点
遇到节点是文本类型的，watch将节点的`textContent`属性和数据相联系。`textContent`需要先用正则对模板语法处理，找到vmKey。当数据改变，整个`textContent`区域渲染为最新的数据。
```
if (node.nodeType == 3){
    // 文本类型
    let reg = /\{\{(.*)\}\}/g;
    console.log(node);
    node.textContent = node.textContent.replace(reg, (match, vmKey)=>{
      vmKey = vmKey.trim();
      if (this.hasOwnProperty(vmKey)){
        let watcher = new Watch(this, vmKey, node, 'textContent');
        if (this.$watchEvent[vmKey]){
          this.$watchEvent[vmKey].push(watcher);
        } else {
          this.$watchEvent[vmKey] = [];
          this.$watchEvent[vmKey].push(watcher);
        }
      }
      return this[vmKey]
    })
}
```

<!--### 生命周期 -->
<!--##### beforeCreate 和 created-->
<!--```-->
<!--update(){-->
<!--if (typeof options.beforeUpdate === 'function'){-->
<!--  options.beforeUpdate.call(this);-->
<!--}-->
<!--this.node[this.attr] = this.vm[this.key]; -->
<!--if (typeof options.updated === 'function'){-->
<!--  options.updated.call(this);-->
<!--}-->
<!--}-->
<!--```-->

这样，一个简易的vue框架完成了。


[代码地址](http://teletubby.github.io)
