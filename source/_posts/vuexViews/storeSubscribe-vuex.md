# subscribe
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#subscribe)

```js{5,9}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {}
  ……
  subscribe(fn, options) {//订阅mutation,就是在this._subscribers装入此fn
    return genericSubscribe(fn, this._subscribers, options)
  }
}
//执行订阅装入操作
function genericSubscribe(fn, subs, options) {
  if (subs.indexOf(fn) < 0) {
    options && options.prepend
      ? subs.unshift(fn)
      : subs.push(fn)
  }
  return () => {//返回函数，执行后是停止订阅
    const i = subs.indexOf(fn)
    if (i > -1) {
      subs.splice(i, 1)
    }
  }
}
```
从官网的介绍了解到，用户可以通过 `this.$store.subscribe(fn, options)` 方法传入函数，这个函数的执行是在 `mutation` 执行后触发。比如我们希望某个 `mutation` 执行后做一些事情，这个方法就很有用。如果希望取消触发：
```js
const cancel = this.$store.subscribe(fn, options)
cancel()
```