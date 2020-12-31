---
title: Module
author: 张凯强
date: 2020-12-31 00:00:00
index_img: https://rmt.dogedoge.com/fetch/fluid/storage/actions-deploy/cover.png?w=480&fmt=webp
category: vuex
tags:
  - vue全家桶
  - vuex
excerpt: 二春子的Module
---

# Module
创建模块的 `class`，创建的模块属性：
+ runtime: 是否是运行时创建的模块标识
+ _children：存储子模块的对象
+ _rawModule：原来用户定义的模块信息
+ state: 此模块处理过的 `state`。从代码可以看出，用户可以定义 `state` 为对象或者函数，最终都会处理成对象的形式

其他一些方法从注释中很容易明白是干什么的
```js{12-22}
import { forEachValue } from '../util'

// module: {//例子
//   _children: {
//     模块A: {……},
//     模块B: {……},
//   }
// }
// Base data struct for store's module, package with some attribute and method
export default class Module {
  constructor(rawModule, runtime) {
    //运行时标记是否为动态模块，注意：通过外部方法registerModule注册的是动态模块
    //unregisterModule卸载模块方法，也只能卸载动态模块（只卸载runtime为true的）
    this.runtime = runtime
    // Store some children item
    this._children = Object.create(null)//装子模块的对象
    // Store the origin module object which passed by programmer
    this._rawModule = rawModule //未处理的当前模块
    const rawState = rawModule.state // 未处理的 state

    // Store the origin module's state所以这里可以看出state既可以写成对象也可以是函数
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }

  get namespaced() {//获取模块的命名空间定义标识
    return !!this._rawModule.namespaced
  }

  addChild(key, module) {//给此模块加子模块
    this._children[key] = module
  }

  removeChild(key) {//删除模块
    delete this._children[key]
  }

  getChild(key) {//根据key获取子模块的实体
    return this._children[key]
  }

  hasChild(key) {//是否有此模块
    return key in this._children
  }

  update(rawModule) {//更新模块
    this._rawModule.namespaced = rawModule.namespaced
    if (rawModule.actions) {
      this._rawModule.actions = rawModule.actions
    }
    if (rawModule.mutations) {
      this._rawModule.mutations = rawModule.mutations
    }
    if (rawModule.getters) {
      this._rawModule.getters = rawModule.getters
    }
  }

  forEachChild(fn) {//对模块的子模块遍历,通过fn函数处理
    forEachValue(this._children, fn)
  }

  forEachGetter(fn) {//对模块的getters遍历,通过fn函数处理
    if (this._rawModule.getters) {
      forEachValue(this._rawModule.getters, fn)
    }
  }

  forEachAction(fn) {//对模块的actions遍历,通过fn函数处理
    if (this._rawModule.actions) {
      forEachValue(this._rawModule.actions, fn)
    }
  }

  forEachMutation(fn) {//对模块的mutations遍历,通过fn函数处理
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn)
    }
  }
}

```