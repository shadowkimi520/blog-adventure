# $route 添加参数
完整代码分支 [stage-1](https://github.com/shengrongchun/parse-vue-router)

假如我们在项目中执行了以下代码
```js
this.$router.push({
  name: 'Bar',params: {age:23},query: {sex: '男'}
})
//或者
<router-link :to={name: 'Bar',params: {age:23},query: {sex: '男'}} />
```
如果路由跳转成功的话，`this.$route` 应该会是这样
```js
$route: {
  path: '/bar',
  name: 'Bar',
  params: {age:23},
  query: {sex: '男'},
  ……
}
```
我们看看路由创建工厂方法 `createRoute` 怎么添加这些参数的
::: tip 注意
假如目前 `location` 是具有 `params,hash,query` 这些值的
:::
```js{10,12,22,25,27,29,55}
import { stringifyQuery } from './query'
export function createRoute(
  record,
  location,
  redirectedFrom,//目前没用到先不看
  router //vueRouter 实例对象
) {
  // 怎么字符串化 query 可以在用户路由配置数据中自定义
  // 官方说明：https://router.vuejs.org/zh/api/#parsequery-stringifyquery
  const stringifyQuery = router && router.options.stringifyQuery
  //
  let query = location.query || {}
  try {
    query = clone(query) // clone方法在下面有定义
  } catch (e) { }
  //
  const route = {
    name: location.name || (record && record.name),//当前路由的名称，如果有的话
    meta: (record && record.meta) || {},//meta元数据，如果有的话
    //字符串，对应当前路由的路径，总是解析为绝对路径，如 "/foo/bar"
    path: location.path || '/',
    hash: location.hash || '',//当前路由的 hash 值 (带 #) ，如果没有 hash 值，则为空字符串
    //一个 key/value 对象，表示 URL 查询参数。例如，对于路径 /foo?user=1，
    //则有 $route.query.user == 1，如果没有查询参数，则是个空对象。
    query,
    //一个 key/value 对象，包含了动态片段和全匹配片段，如果没有路由参数，就是一个空对象
    params: location.params || {},
    //包含查询参数和 hash 的完整路径
    fullPath: getFullPath(location, stringifyQuery),//getFullPath方法在下面有定义
    //当前路由匹配的组件
    matched: { components: record ? record.components : {} }
  }
  //
  return Object.freeze(route)//冻结对象，不让其修改
}
//
function clone(value) {//递归克隆
  if (Array.isArray(value)) {
    return value.map(clone)
  } else if (value && typeof value === 'object') {
    const res = {}
    for (const key in value) {
      res[key] = clone(value[key])
    }
    return res
  } else {
    return value
  }
}
//获取完整路径包括：path query hash
function getFullPath(
  { path, query = {}, hash = '' },
  _stringifyQuery
) {
  const stringify = _stringifyQuery || stringifyQuery//如果没有自定义，使用默认的
  return (path || '/') + stringify(query) + hash
}
```
代码上都有很详细的注释，我们还额外增加了一个 `fullPath` 完整路径变量。他是 `path+query+hash`
::: tip
定义的参数基本都是从 `location` 而来。`55` 行代码中有默认的字符串化 `query` 的方法 `stringifyQuery` （当用户没有自定义字符串化方法时使用）
:::
我们直接进入[stringifyQuery方法查看](/routerViews/util-router.html#stringifyquery)

整个项目唯一用到 `createRoute` 的地方就是 `createMatcher` 这里
```js{11,19-21,24,37}
export function createMatcher(
  routes,
  router
) {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  //
  function match(
    raw,
    currentRoute
  ) {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location
    if (name) {
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {//警告 没有发现此name的record
        warn(record, `Route with name '${name}' does not exist`)
      }
      if (!record) return _createRoute(null, location) //没有，直接创建一个默认的route
      if (typeof location.params !== 'object') {// params 不是对象就设置成空对象
        location.params = {}
      }
      return _createRoute(record, location)
    } else if (location.path) {
      location.params = {} // 这句代码，让在有path的时候无视了params
      //注意：pathMap只会有 '':  没有 '/':
      const record = pathMap[path === '/' ? '' : path]
      return _createRoute(record, location)
    }
    // no match
    return _createRoute(null, location)
  }
  //
  function _createRoute(
    record,
    location,
  ) {
    return createRoute(record, location)
  }
  //
  return {
    match
  }
}
```
::: tip 解析
相比之前的代码，这里加了些对 `params` 的处理还有一些警告。`24` 行代码非常重要，这是为什么当你 `push({path: xxx, params: {……}})` 的时候，`params` 参数无效的原因
:::
`normalizeLocation` 方法之前只解析 `path` 的，此时我们还要加上解析 `params,hash,query` 等等。可以[查看normalizeLocation](/routerViews/util-router.html#normalizelocation)。

