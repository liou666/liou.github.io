---
layout: post
title: 实现一个mini-vue
subtitle: 复习vue响应式原理
date: 2021-4-5
author:
header-img:
catalog: true
tags:
  - vue
typora-root-url: ..
---

### 手写 vue 框架

> 复习 vue 的响应式原理，基于发布订阅者模式基本实现了 vue 的数据更新视图的功能，但很多细节还未完善。

##### 先上一个经典的图

![](https://ftp.bmp.ovh/imgs/2021/04/9710f7dad0a3b48b.jpg)

`data`中的数据通过`Observer`加入到响应式系统中，通过对象的 `Object.Property` 方法为 `data` 中的每个属性添加 `get` 和 `set`，**data 中的每个属性各对应一个 `Dep` 对象**，当数据发生变化时，会走到 `set` 中触发的 `dep.notify` 方法，遍历 `subs` 中的 `watch` 集合并触发 每个 watch 的 `updata` 方法进行视图更新。
`el` 的值则会通过 `Compile` 进行模板指令解析，在解析的过程中订阅数据变化，并绑定更新函数，**也就是创建一个个的 `watch` 实例**。

##### html 测试代码

```html
<body>
  <div id="app"><input type="text" />{{name}}</div>
  <script>
    const app = new Vue({
      el: "#app",
      data: {
        name: "liou",
        age: 18,
      },
    });
  </script>
</body>
```

##### mini-vue 代码

```javascript
/**
 * 创建mvvm模型
 * @param {options}
 */
class Vue {
  constructor(options) {
    this.$el = options.el;
    this.$data = options.data;

    //数据劫持（将data中的数据添加到响应式系统中）
    new Observer(this.$data, this);

    //代理this.$data中的数据
    Object.keys(this.$data).forEach((key) => this._proxy(key));

    //解析指令 （解析el模板）
    new Compile(this.$el, this);
  }

  _proxy(key) {
    Object.defineProperty(this, key, {
      enumerable: true,
      configurable: true,
      get() {
        return this.$data[key];
      },
      set(newValue) {
        this.$data[key] = newValue;
      },
    });
  }
}

/**
 * 编译指令
 * @param {el,vm}
 */
class Compile {
  constructor(el, vm) {
    this.el = document.querySelector(el);
    this.vm = vm;
    const f = this._createFragment(this.el);
    this.el.appendChild(f);
  }
  _createFragment(node) {
    const fragment = document.createDocumentFragment();
    let childNode;
    //注：所有DOM节点插入方法都会自动从旧位置删除该节点
    while ((childNode = node.firstChild)) {
      this._compile(childNode); //去编译节点内容
      fragment.appendChild(childNode);
    }
    return fragment;
  }
  _compile(node) {
    if (node.nodeType === 3) {
      //文本节点
      let reg = /\{\{(.*)\}\}/g;
      if (reg.test(node.nodeValue)) {
        const text = RegExp.$1.trim();
        new Watch(node, text, this.vm); //新建观察者（订阅数据变化，绑定更新函数）
      }
    }
    if (node.nodeType === 1) {
      //标签节点
    }
  }
}

/**
 * 数据劫持
 * @param {data,vm}
 */
class Observer {
  constructor(data, vm) {
    this.vm = vm;
    Object.keys(data).forEach((key) =>
      this.objectReactive(data, key, data[key])
    );
  }
  //为data中每个属性添加get和set
  objectReactive(obj, key, value) {
    const dep = new Dep();
    Object.defineProperty(obj, key, {
      get() {
        Dep.target && dep.addSub(Dep.target);
        return value;
      },
      set(newValue) {
        if (value === newValue) return;
        value = newValue;
        dep.notify();
      },
    });
  }
}

/**
 * 依赖收集器
 */
class Dep {
  constructor() {
    this.subs = [];
  }
  //添加依赖
  addSub(watch) {
    this.subs.push(watch);
  }
  //通知数据更新
  notify() {
    this.subs.forEach((w) => w.update());
  }
}

/**
 * 依赖（观察者）
 * @param {node, text, vm} 节点 解析的文本 vue实例
 */
class Watch {
  constructor(node, text, vm) {
    this.node = node;
    this.text = text;
    this.vm = vm;
    Dep.target = this;
    this.update();
    Dep.target = null;
  }
  update() {
    console.log("数据更新");
    this.node.nodeValue = this.vm.$data[this.text];
    // this.vm.$data[this.text]会走到get方法，此时给dep添加watch刚好合适
  }
}
```
