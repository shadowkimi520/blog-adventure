# plugins
完整代码分支 [stage-2](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#plugins)

源码也很简单
```js{4,9}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    const {
      plugins = [],//用户定义插件
      strict = false// 用户定义是否严格模式，在严格模式下，任何 mutation 处理函数以外修改 Vuex state 都会抛出错误
    } = options
    …………
    // apply plugins
    plugins.forEach(plugin => plugin(this))
  }
}
```
用户定义的插件，类型是数组。内部遍历执行数组里面的函数，函数的入参是 `store`