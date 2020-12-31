# subscribeAction
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#subscribeaction)

```js{5,6,10}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {}
  ……
  subscribeAction(fn, options) {//订阅action
    const subs = typeof fn === 'function' ? { before: fn } : fn
    return genericSubscribe(subs, this._actionSubscribers, options)
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
`this.$store.subscribeAction` 是对于 `action` 执行触发而言。不过这个函数传入的参数 `fn` 稍微复杂一点。基于 `action` 异步的特性：
+ `fn` 是函数类型，会直接当作是在 `action` 执行之前执行此 `fn` 
+ `fn` 是对象类型，可以在对象里面定义在 `action` 执行前后需要执行的函数，或者 `action` 执行报错后需要执行的函数
```js
const fn = {
  before: fn1,
  after: fn2,
  error: fn3
}
```