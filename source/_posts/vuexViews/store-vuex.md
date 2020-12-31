# $store
[官网介绍](https://vuex.vuejs.org/zh/guide/)

完整代码分支 [stage-0](https://github.com/shengrongchun/parse-vue-vuex)

## 从一个简单例子开始
```js{7,9,17}
import Vue from 'vue'
import App from './App.vue'
import vuex from '../vuex'

Vue.config.productionTip = false
//
Vue.use(vuex)
//
const store = new vuex.Store({
  state: {},
  mutations: {},
  actions: {},
  getters: {},
})

new Vue({
  store,
  render: h => h(App),
}).$mount('#app')
```
首先我们需要搞清楚 `vuex` 是如何做到每一个组件里都有 `this.$store` 对象的

`vuex` 源码开始文件 `index.js`
```js
import { Store, install } from './store'

// 这种模式 vuex.Store
export default {
  Store,
  install,
  version: '__VERSION__',
}
```
显然，从上面的例子我们得出，引入的 `vuex` 是必须要有 `install` 方法的，因为使用了 `Vue.use(vuex)`。还应该要有 `Store Class` 因为使用了 `new vuex.Store`。接下来我们看看 `install Store` 到底做了什么
```js
import applyMixin from './mixin'
import { assert } from './util'
let Vue // bind on install
export class Store { // this--> vue实例中的 this.$store
  constructor() {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue) //window上有Vue会自动安装的
    }

    if (__DEV__) {//开发环境会有一些警告。比如创建store实例前需要执行Vue.use(Vuex)等
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `store must be called with the new operator.`)
    }
  }
}
//
export function install(_Vue) {//插件安装
  if (Vue && _Vue === Vue) {//安装过了
    if (__DEV__) {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```
首先我们看 `install` 方法，很容易看懂，`Vue.use` 会执行这个 `install` 方法。当然也判断了是否是重复安装，第一次安装会执行 `applyMixin` 方法

再看 `Store Class`，它也做了相应判断处理，比如 `window` 上挂载了 `Vue`，并且没有执行 `Vue.use` 会自动安装

`applyMixin`

```js{4,19-28}
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])
  if (version >= 2) {//大于等于2版本
    Vue.mixin({ beforeCreate: vuexInit })
  } else {//版本小于1的处理方式
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }
  /**
   * Vuex init hook, injected into each instances init hooks list.
   */
  function vuexInit() {//这样在整个项目的this实例中都可以获取到$store
    const options = this.$options
    // store injection
    if (options.store) {//此时的this是根实例
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```
我们只解析 `Vue` 大于等于 `2` 的版本。如果看过 `vue-router` 系列解析文章就会很容易看懂 `vuexInit` 这个方法是在做什么。

+ 第一步，我们使用 `Vue` 的混入功能，让每个组件初始化的时候都会执行我们定义的 `vuexInit` 方法
+ `vuexInit` 方法中可以拿到每个组件实例，可以在实例上定义 `$store`
+ 第二步，`$store` 指向问题，从源码可以看出它应该指向根实例的 `options.store`
+ 第三步，找到 `Vue` 的根实例，然后找到 `options.store` 赋值给 `$store`
+ 第四步，由于父子组件遍历的顺序关系，子组件的实例只要获取父组件实例的 `$store` 即可

```js{1,8}
const store = new vuex.Store({
  state: {},
  mutations: {},
  actions: {},
  getters: {},
})
new Vue({
  store,
  render: h => h(App),
}).$mount('#app')
```
::: warning 注意
判断 `options.store` 是根实例的依据是上面的代码
:::

## 测试 $store
我们测试下 `this.$store` 是否是 `Store Class` 实例对象
```js{16}
export class Store { // this--> vue实例中的 this.$store
  constructor() {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue) //window上有Vue会自动安装的
    }

    if (__DEV__) {//开发环境会有一些警告。比如创建store实例前需要执行Vue.use(Vuex)等
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `store must be called with the new operator.`)
    }
    //
    this.test = '这个this就是在组件里使用的 this.$store'
  }
}
```
```vue {12}
<template>
  <div>vuex {{test}}</div>
</template>
<script>
export default {
  data() {
    return {
      test: this.$store.test
    }
  },
  created() {
    console.log('$store', this.$store)
  }
}
</script>
```
![store](./img/store.jpg)