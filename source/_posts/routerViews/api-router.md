# 完善 `api`
[官网介绍](https://router.vuejs.org/zh/api/#router-%E5%AE%9E%E4%BE%8B%E6%96%B9%E6%B3%95)

完整代码分支 [stage-12](https://github.com/shengrongchun/parse-vue-router)

`api` 的完善主要是对于 `$router` 对象而言，可能大多数方法和属性都在系列文章中解析过。现在我们要列出一些还未解析的 `api`:
### router.currentRoute
获取当前路由
```js
get currentRoute() {// 获取当前路由
  return this.history && this.history.current
}
```

### router.app.$once('hook:destroyed')
```js
// set up app destroyed handler
// https://github.com/vuejs/vue-router/issues/2639
app.$once('hook:destroyed', () => {
  // clean out app from this.apps array once destroyed
  const index = this.apps.indexOf(app)
  if (index > -1) this.apps.splice(index, 1)
  // ensure we still have a main app or null if no apps
  // we do not release the router so it can be reused
  if (this.app === app) this.app = this.apps[0] || null

  if (!this.app) {
    // clean up event listeners
    // https://github.com/vuejs/vue-router/issues/2341
    this.history.teardownListeners()
  }
})
```
销毁指定 `app` 路由事件的方法

### router.go
```js
go(n) {
  this.history.go(n)
}
```
### router.back
```js
back() {
  this.go(-1)
}
```
### router.forward
```js
forward() {
  this.go(1)
}
```
### html5 中的 history.go
```js
go(n) {
  window.history.go(n)
}
```
### hash 中的 history.go
```js
go(n) {
  window.history.go(n)
}
```
### abstract 中的 history.go
```js{6}
go(n) {
  const targetIndex = this.index + n
  if (targetIndex < 0 || targetIndex >= this.stack.length) {
    return
  }
  const route = this.stack[targetIndex]
  this.confirmTransition(
    route,
    () => {
      this.index = targetIndex
      this.updateRoute(route)
    },
    err => {
      if (isRouterError(err, NavigationFailureType.duplicated)) {
        this.index = targetIndex
      }
    }
  )
}
```
`abstract` 一般用与非浏览器端，其 `go` 方法看代码也很简单，就是从存放路由的 `stack` 数组中获取我们需要的路由对象。然后跳转到此路由

### router.getMatchedComponents
返回目标位置或是当前路由匹配的组件数组 (是数组的定义/构造类，不是实例)。通常在服务端渲染的数据预加载时使用
```js
getMatchedComponents(to) {
  const route = to
    ? to.matched
      ? to
      : this.resolve(to).route
    : this.currentRoute
  if (!route) {
    return []
  }
  return [].concat.apply([], route.matched.map(m => {
    return Object.keys(m.components).map(key => {
      return m.components[key]
    })
  }))
}
```
### router.addRoutes
动态添加更多的路由规则。参数必须是一个符合 routes 选项要求的数组
```js{2}
addRoutes(routes) {
  this.matcher.addRoutes(routes)
  if (this.history.current !== START) {
    this.history.transitionTo(this.history.getCurrentLocation())
  }
}
```
第二行代码是关键

matcher.addRoutes
::: tip
第一行代码很关键，把之前创建好的 `pathList, pathMap, nameMap` 当做数组参数传入。使得新增的路由配置信息放入其中，起到动态添加路由规则的功能
:::
```js{1}
const { pathList, pathMap, nameMap } = createRouteMap(routes)
function addRoutes(routes) {// 动态添加路由配置信息函数
  createRouteMap(routes, pathList, pathMap, nameMap)
}
```
原来的 createRouteMap
```js
export function createRouteMap(
  routes, //传来的路由配置信息
) {
  const pathList = [] //创建空数组
  const pathMap = Object.create(null)//创建空对象
  const nameMap = Object.create(null)//创建空对象
  ……………………
}
```
改变后的 createRouteMap
```js{7}
export function createRouteMap(
  routes, //传来的路由配置信息
  oldPathList,
  oldPathMap,
  oldNameMap
) {
  const pathList = oldPathList || [] //创建空数组
  const pathMap = oldPathMap || Object.create(null)//创建空对象
  const nameMap = oldNameMap || Object.create(null)//创建空对象
  ……………………
}
```
明显可以看出如果有 `oldPathList` 那么 新创建的 `pathList` 就是 `oldPathList`