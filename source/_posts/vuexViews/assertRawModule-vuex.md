---
title: assertRawModule
author: 张凯强
date: 2020-12-31 00:00:00
index_img: https://rmt.dogedoge.com/fetch/fluid/storage/actions-deploy/cover.png?w=480&fmt=webp
category: vuex
tags:
  - 部署
  - 示例
  - Hexo
excerpt: 二春子的assertRawModule
---

# assertRawModule
```js{17}
//从Assert可以得出：mutations和getters里面定义的只能是函数，而actions可以是函数或者含有handler函数的对象
//assertRawModule作用就是判断模块中定义的类型是否正确，在dev环境下会警告
const functionAssert = {
  assert: value => typeof value === 'function',
  expected: 'function'
}
const objectAssert = {
  assert: value => typeof value === 'function' ||
    (typeof value === 'object' && typeof value.handler === 'function'),
  expected: 'function or object with "handler" function'
}
const assertTypes = {
  getters: functionAssert, //期望getters 是函数
  mutations: functionAssert,// 期望 mutations 是函数
  actions: objectAssert // 期望actions是函数或者是对象并且对象的 handler 是函数
}
function assertRawModule(path, rawModule) {
  Object.keys(assertTypes).forEach(key => {//key: getters mutations actions
    if (!rawModule[key]) return // 检测模块没有定义 getters mutations actions 直接return

    const assertOptions = assertTypes[key] //获取相关类型的 Assert: {assert,expected}
    // mutations: {//rawModule[key]
    //   name：(state)=>{
    //     ……
    //   }
    // }
    //forEachValue 与 assert 可以在 工具-方法 中查看
    forEachValue(rawModule[key], (value, type) => {//value是如：(state)=> {}，type如：name
      assert(
        assertOptions.assert(value),
        makeAssertionMessage(path, key, type, value, assertOptions.expected)
      )
    })
  })
}
//path:[]装模块名称的数组 key:getters/mutations/actions type:相关(getter/actions/mutations)里面定义的函数名称
//value: 里面定义的函数体
function makeAssertionMessage(path, key, type, value, expected) {
  let buf = `${key} should be ${expected} but "${key}.${type}"`
  if (path.length > 0) {
    buf += ` in module "${path.join('.')}"`
  }
  buf += ` is ${JSON.stringify(value)}.`
  return buf
}
```
上述代码加上注释非常容易看懂，此方法的作用主要是约束用户定义的 `getters/mutations/actions` 类型是否正确，从 `Assert` 可以得出：`mutations` 和 `getters` 里面定义的只能是函数，而 `actions` 可以是函数或者含有 `handler` 函数的对象。举例：
```js
{
  actions: {
    name: 123,//类型错误
    sex: {},//类型错误
    age() {},//类型正确
    address: {//类型正确
      handler() {}
    }
  }
}
```
::: tip 提示
`forEachValue` 与 `assert` 可以在 `工具-方法` 中查看
:::