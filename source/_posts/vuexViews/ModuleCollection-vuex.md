---
title: ModuleCollection
author: 张凯强
date: 2020-12-31 00:00:00
index_img: https://rmt.dogedoge.com/fetch/fluid/storage/actions-deploy/cover.png?w=480&fmt=webp
category: vuex
tags:
  - vue全家桶
  - vuex
excerpt: 二春子的ModuleCollection
---

# ModuleCollection
`ModuleCollection` 从创建根模块对象 `this.root` 开始，通过子模块对象放入父模块的 `_children` 中。递归遍历最终收集所有的模块

```js{4,13,15,17}
export default class ModuleCollection {
  constructor(rawRootModule) {//options
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)//runtime是标识创建的模块是运行时创建的，还是初始化创建的
  }
  //
  get(path) {
    return path.reduce((module, key) => {//reduce数组为空，直接返回初始值 this.root
      return module.getChild(key)//每次去寻找模块时都是根据模块的名字key到父模块的__children去寻找
    }, this.root)//根据reduce的特性，module一开始是this.root,下次就是module.getChild(key)的返回值
  }
  //注册收集模块，父模块把所有子模块收集到自己的__children里面
  register(path, rawModule, runtime = true) {
    if (__DEV__) {//dev环境下判断当前模块下定义的actions,mutations,getters类型是否正确
      assertRawModule(path, rawModule)//辅助目录下查看
    }
    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {//如果path为空数组，此时的模块为根模块
      this.root = newModule//我们已经把根模块存到root中
    } else {
      const parent = this.get(path.slice(0, -1))//根据模块名称获取此时模块的父模块
      parent.addChild(path[path.length - 1], newModule)//再把此时模块装入父模块的_children中
    }

    // register nested modules 注册模块的子模块
    if (rawModule.modules) {//path在过程中启到标识模块层级的作用，如path: [a,ab]说明模块a,以及a里面的模块ab
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)//path: [a,ab]
      })
    }
  }
}
```
`path` 是一个数组，装入的是模块名字，如：`path=[模块A,模块B]`，它的含义是根模块下面有个`模块A`，`模块A`下面有个`模块B`。 这样的 `path` 结构可以迅速获取到父模块的名字，为模块装入父模块，获取父模块提供帮助

`runtime` 是标识创建的模块是运行时产生的，还是初始化时候产生的。`constructor` 里面执行的当然是初始化阶段，当我们在其他地方调用 `register` 注册动态模块的时候，默认就是运行时创建模块

+ `15` 行代码是在非生产环境下，需要对用户定义的 `actions,mutations,getters` 等进行定义类型检测,如果类型定义错误会发出错误警告。具体详情请看 `辅助` 下面的 `assertRawModule`
+ `17` 行代码是对用户定义的模块配置信息进行对象模块化。具体详情请看 `辅助` 下面的 `Module`

经过 `17` 行代码后，我们创建了基于用户定义模块信息的新模块对象 `newModule`。对象包括：`runtime _children _rawModule state ` 还有一些方法。
+ 从 `18` 行到 `23` 行是模块装入的过程，通过 `path` 来判断出根模块，然后赋给 `root`。其他模块再通过 `path get` 获取模块的父模块，最终装入父模块的过程。过程中用到的方法 `getChild addChild` 可以在 `辅助` 下面的 `Module` 中查看
+ `26-30` 行代码是递归遍历所有模块的过程

## 总结
除了根模块之外，子模块都是装入父模块的 `_children` 对象里面。对象的 `key` 是模块名，如：
```js{6}
this.root = {
  _children: {
    模块A: {……},
    模块B: {
      _children: {
        模块C: {……}
      }
    }
  }
}
```
` new ModuleCollection` 最终返回了实例对象，实例对象中有属性 `root` 结构如上所示。我们返回 `基础` 下面的 `module` 部分