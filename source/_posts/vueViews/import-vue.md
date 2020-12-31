# Vue是什么
## 引入Vue 
我们在通过 `vue` 框架开发页面时，都需要先去引入 `vue` 文件,例如:

```js
import Vue from vue
```
那么返回的 `Vue` 是什么呢？我们还是看看源码到底返回了什么:
```js
function Vue (options) { // 原来Vue就是一个构造函数
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
export default Vue
```
`Vue` 是构造函数，当我们去 `new Vue({...})` 会生成一个实例对象如 `vm`。
## 如何代理 methods方法
我们看看如下代码:
````js
var options = {
  data() {
    return {
      name: '哈啰出行'
    }
  },
  created() {
    this.changeName()
  },
  methods: {
    changeName() {
      this.name = '哈啰出行，我看行'
    }
  }
}
new Vue(options)
````
`this` 就是我们 `new` 出来的对象。但是 `this.changeName` 为什么可以获取到`methods` 里面定义的方法呢？
````js
function initMethods (vm, methods) {//options.methods
  for (const key in methods) {//noop-->()=> {}
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm)
  }
}
````
很显然 `this[key]` 已经被代理到 `methods[key]`
````js
function bind(fn, vm) {
  return fn.bind(vm)
}
````
至于使用 `bind` 函数包裹，是强制改变函数内部的 `this` 为 `vm`
## 如何代理 data 数据
````js
function initData (vm, data) {//options.data
  vm._data = typeof data === 'function'
    ? data.call(vm, vm)
    : data || {}
  let keys  = Object.keys(vm._data);
  proxyKeys(vm, keys)
}
function proxyKeys(vm, keys) {
  while (i--) { // 对每一个key代理
    const key = keys[i]
    proxy(vm, `_data`, key)
  }
}
// 代理 this.xxx --> this._data.xxx
function proxy (target, sourceKey, key) { // vm,_data,key
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
````
如果 `data` 是函数，直接执行 `data` 函数生成对象赋值给 `vm._data`, 如果非函数直接`data` 赋值。所以现在 `vm._data[key] --> data[key]`,然后通过 `proxy`直接 `vm[key]` 代理 `vm._data[key]`
## 小节
还有 `props、computed` 等都是通过类似的方法来代理的。当然初始化的时候会判断名称,如果重复会报错，当然 `props` 的优先级最高。依次为 `props、methods、data、computed、watch`