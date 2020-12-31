# 重定向和别名
[官网介绍](https://router.vuejs.org/zh/guide/essentials/redirect-and-alias.html)

完整代码分支 [stage-6](https://github.com/shengrongchun/parse-vue-router)

## 重定向

重定向也是通过 `routes` 配置来完成，下面例子是从 `/a` 重定向到 `/b`：
```js
const router = new VueRouter({
  routes: [
    { path: '/a', redirect: '/b' }
  ]
})
```
重定向的目标也可以是一个命名的路由：
```js
const router = new VueRouter({
  routes: [
    { path: '/a', redirect: { name: 'foo' }}
  ]
})
```
甚至是一个方法，动态返回重定向目标：
```js
const router = new VueRouter({
  routes: [
    { path: '/a', redirect: to => {
      // 方法接收 目标路由 作为参数
      // return 重定向的 字符串路径/路径对象
    }}
  ]
})
```
首先我们需要解析我们从 `routes` 传来的 `redirect` 参数

### create-route-map
```js{37}
import Regexp from 'path-to-regexp'
import { cleanPath } from './util/path'
import { assert, warn } from './util/warn'
//
export function createRouteMap(
  routes, //传来的路由配置信息
) {
  const pathList = [] //创建空数组
  const pathMap = Object.create(null)//创建空对象
  const nameMap = Object.create(null)//创建空对象
  //遍历 routes 把 route 相关信息放入 pathList, pathMap, nameMap
  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })
  …………
  //
  return {
    pathList,
    pathMap,
    nameMap
  }
}
function addRouteRecord(
  pathList,
  pathMap,
  nameMap,
  route,
  parent,
) {
  …………
  const record = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    name,
    parent,
    redirect: route.redirect, // 有重定向参数
    meta: route.meta || {},
    props: //路由组件传参 有了这个功能，组件可以不再和$route耦合
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props } //非多组件的情况下，就是对默认组件的设置
  }
  …………
}
```
代码很简单，就是在 `record` 上加了重定向的标识，在看创建路由阶段是怎么处理的

### create-matcher
```js{16,23,28,31,35-37}
/*…………*/
export function createMatcher(
  routes,
  router
) {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  function match(
    raw,
    currentRoute,
  ) {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location
    if (name) {
      const record = nameMap[name]
      /*…………*/
      return _createRoute(record, location)
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
  //
  function _createRoute(
    record,
    location,
  ) {
    if (record && record.redirect) { // 如果有重定向
      return redirect(record, location)
    }
    return createRoute(record, location)
  }
  //
  return {
    match
  }
}
/*…………*/
function resolveRecordPath(path, record) {
  return resolvePath(path, record.parent ? record.parent.path : '/', true)
}
```
这里把不相关的代码都省略了，当创建路由时，最终都要执行 `_createRoute` 方法。在此方法中做了判断，如果有重定向配置，就执行 `redirect` 方法。我们看 `redirect` 是怎么实现的

通过官网介绍可以得知，重定向参数会有三种类型，首先是对这三种类型进行解析处理
### redirect
```js{6-16,39-50,53-62}
//重定向方法
function redirect(
  record,
  location
) {
  const originalRedirect = record.redirect // 参数会有三种类型
  // 函数类型:
  // createRoute(record, location, null, router)--> 此to就是函数的参数：原本去哪里的路由对象
  // 如：path: '/a'. redirect: (to)=> {} to就是 '/a' 创建的路由对象
  let redirect = typeof originalRedirect === 'function' 
    ? originalRedirect(createRoute(record, location, null, router)) 
    : originalRedirect
  // 字符串类型直接当做 path 来处理
  if (typeof redirect === 'string') {
    redirect = { path: redirect }
  }
  // 其他类型会警告
  if (!redirect || typeof redirect !== 'object') {
    if (process.env.NODE_ENV !== 'production') {
      warn(
        false, `invalid redirect option: ${JSON.stringify(redirect)}`
      )
    }
    return _createRoute(null, location)
  }
  // 到这里：redirect一定是个对象类型
  const re = redirect
  const { name, path } = re//解析 name path
  let { query, hash, params } = location
  // 如果重定向路由没有 query hash params。就把原跳转路由上相关的加入
  // eslint-disable-next-line no-prototype-builtins
  query = re.hasOwnProperty('query') ? re.query : query
  // eslint-disable-next-line no-prototype-builtins
  hash = re.hasOwnProperty('hash') ? re.hash : hash
  // eslint-disable-next-line no-prototype-builtins
  params = re.hasOwnProperty('params') ? re.params : params

  if (name) {
    // resolved named direct
    const targetRecord = nameMap[name]
    if (process.env.NODE_ENV !== 'production') {
      assert(targetRecord, `redirect failed: named route "${name}" not found.`)
    }
    return match({//
      _normalized: true,
      name,
      query,
      hash,
      params
    }, undefined)
  } else if (path) {
    // 1. resolve relative redirect 解析相对跳转path
    const rawPath = resolveRecordPath(path, record)
    // 2. resolve params 再追加上params
    const resolvedPath = fillParams(rawPath, params, `redirect route with path "${rawPath}"`)
    // 3. rematch with existing query and hash
    return match({
      _normalized: true,
      path: resolvedPath,
      query,
      hash
    }, undefined)
  } else {
    if (process.env.NODE_ENV !== 'production') {
      warn(false, `invalid redirect option: ${JSON.stringify(redirect)}`)
    }
    return _createRoute(null, location)
  }
}
```
如果一切顺利，`redirect` 对象应该可以解析出 `path` 或者还有 `name` 。我们直接从 `38` 行代码开始看。
如果有 `name`。按重定向的目的来说，我们应该直接创建重定向的路由。比如：
```js
if(name) {
  const targetRecord = nameMap[name]
  const location = { 
    name,
    query,
    hash,
    params}
  return createRoute(targetRecord, location)
}
```
然而最终的代码是这样的
```js{7-13}
if (name) {
  // resolved named direct
  const targetRecord = nameMap[name]
  if (process.env.NODE_ENV !== 'production') {// targetRecord不存在会报错
    assert(targetRecord, `redirect failed: named route "${name}" not found.`)
  }
  return match({
    _normalized: true,
    name,
    query,
    hash,
    params
  }, undefined)
}
```
为什么又重新去执行 `match` 方法了呢？
+ 如果直接执行 `createRoute` ，动态路由的 `path` 并没有处理成最终有效的 `path`，相对当前路由 `params` 也要处理
+ 如果直接执行 `createRoute` ，并不能解决重定向后的路由配置中可能还会有重定向的配置

因此，倒不如直接执行 `match` 方法，传入重定向 `location` 的参数


我们再看看 `path` 判断的部分
```js
else if (path) {
  // 1. resolve relative redirect 解析相对跳转 path
  const rawPath = resolveRecordPath(path, record)
  // 2. resolve params 再追加上params
   //请到工具公共方法查看
  const resolvedPath = fillParams(rawPath, params, `redirect route with path "${rawPath}"`)
  // 3. rematch with existing query and hash
  return match({
    _normalized: true,
    path: resolvedPath,
    query,
    hash
  }, undefined)
}
function resolveRecordPath(path, record) {
  //请到工具公共方法查看
  return resolvePath(path, record.parent ? record.parent.path : '/', true)
}
```
+ 首先判断 `path` 是否是以 `/` 开头，不是则配置相对 `path` 。比如 `path = 'a/b'` , `record.parent.path = /c`。解析后的 `path` 应该是： `c/a/b`
+ `fillParams` 是处理动态路由 `path` 的，比如 `path='/a/:id' params: {id:'b'}`， 最终解析出的 `resolvedPath: '/a/b'`

`createRoute` 方法中我们还要给路由对象添加一个 `redirectedFrom` 属性
```js{5,34-36}
import { stringifyQuery } from './query'
export function createRoute(
  record,
  location,
  redirectedFrom,//这里
  router //vueRouter 实例对象
) {
  // 怎么字符串化 query 可以在用户路由配置数据中自定义
  // 官方说明：https://router.vuejs.org/zh/api/#parsequery-stringifyquery
  const stringifyQuery = router && router.options.stringifyQuery
  //
  let query = location.query || {}
  try {
    query = clone(query) // clone方法在下面有定义
    // eslint-disable-next-line no-empty
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
    //一个数组，包含当前路由的所有嵌套路径片段的路由记录
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  //
  return Object.freeze(route)//冻结对象，不让其修改
}
/*…………*/
//获取完整路径包括：path query hash
function getFullPath(
  { path, query = {}, hash = '' },
  _stringifyQuery
) {
  const stringify = _stringifyQuery || stringifyQuery//如果没有自定义，使用默认的
  return (path || '/') + stringify(query) + hash
}
```
我们看看最终的 `create-matcher`
```js{17,46,58,63,111,123,135,140}
/* eslint-disable no-prototype-builtins */
import { resolvePath } from './util/path'
import { assert, warn } from './util/warn'
import { createRoute } from './util/route'
import { fillParams } from './util/params'
import { normalizeLocation } from './util/location'
import { createRouteMap } from './create-route-map'

export function createMatcher(
  routes,
  router
) {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  function match(
    raw,
    currentRoute,
    redirectedFrom
  ) {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location
    if (name) {
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {//警告 没有发现此name的record
        warn(record, `Route with name '${name}' does not exist`)
      }
      if (!record) return _createRoute(null, location) //没有，直接创建一个默认的route
      //通过正则来收集动态path参数名 如： /a/:username/:userid   paramNames: [username, userid]
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)
      //
      if (typeof location.params !== 'object') {// params 不是对象就设置成空对象
        location.params = {}
      }
      //如果location的params里面没有如username, userid，那么看看当前路由上的params有没有，有就加进来。
      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }
      // 把params填充到path，如record.path: /a/:username/:userid; params: {username:vue,userid:router};
      //那么最终的path: /a/vue/router
      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location, redirectedFrom)
    } else if (location.path) {
      location.params = {} // 这句代码，让在有path的时候无视了params
      // 动态路由这里需要改造 直接的pathMap[location.path]是获取不到 record的
      // 如：pathMap:{ '/man/:id': record } location.path: /man/123
      // 我们可以轮询pathMap里的record ,通过record.regex来匹配 location.path，如果匹配上就获取到了匹配的record
      // 当然这里是会获取最先匹配上的 即使：pathMap: { '/man/:id': record1, '/man/:user': record2 } 
      // location.path: /man/123 只会匹配到 record1
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location, redirectedFrom)
  }
  //重定向方法
  function redirect(
    record,
    location
  ) {
    const originalRedirect = record.redirect//参数会有三种类型
    let redirect = typeof originalRedirect === 'function'//函数
      ? originalRedirect(createRoute(record, location, null, router)) // createRoute(record, location, null, router)--> to
      : originalRedirect

    if (typeof redirect === 'string') {//字符串
      redirect = { path: redirect }
    }

    if (!redirect || typeof redirect !== 'object') {//非三种类型警告
      if (process.env.NODE_ENV !== 'production') {
        warn(
          false, `invalid redirect option: ${JSON.stringify(redirect)}`
        )
      }
      return _createRoute(null, location)
    }

    const re = redirect
    const { name, path } = re//解析 name path
    let { query, hash, params } = location
    // 如果重定向路由没有 query hash params。就把原跳转路由上的相关加入
    // eslint-disable-next-line no-prototype-builtins
    query = re.hasOwnProperty('query') ? re.query : query
    // eslint-disable-next-line no-prototype-builtins
    hash = re.hasOwnProperty('hash') ? re.hash : hash
    // eslint-disable-next-line no-prototype-builtins
    params = re.hasOwnProperty('params') ? re.params : params

    if (name) {
      // resolved named direct
      const targetRecord = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        assert(targetRecord, `redirect failed: named route "${name}" not found.`)
      }
      return match({//
        _normalized: true,
        name,
        query,
        hash,
        params
      }, undefined, location)
    } else if (path) {
      // 1. resolve relative redirect 解析相对跳转path
      const rawPath = resolveRecordPath(path, record)
      // 2. resolve params 再追加上params
      const resolvedPath = fillParams(rawPath, params, `redirect route with path "${rawPath}"`)
      // 3. rematch with existing query and hash
      return match({
        _normalized: true,
        path: resolvedPath,
        query,
        hash
      }, undefined, location)
    } else {
      if (process.env.NODE_ENV !== 'production') {
        warn(false, `invalid redirect option: ${JSON.stringify(redirect)}`)
      }
      return _createRoute(null, location)
    }
  }
  //
  function _createRoute(
    record,
    location,
    redirectedFrom
  ) {
    if (record && record.redirect) { // 如果有重定向
      return redirect(record, redirectedFrom || location)
    }
    return createRoute(record, location, redirectedFrom)
  }
  //
  return {
    match
  }
}
function matchRoute(
  regex,
  path,
  params
) {
  //
  const m = path.match(regex)
  if (!m) {//没有匹配上
    return false
  } else if (!params) { //?
    return true
  }
  for (let i = 1, len = m.length; i < len; ++i) {
    const key = regex.keys[i - 1]
    const val = typeof m[i] === 'string' ? decodeURIComponent(m[i]) : m[i]
    if (key) {
      // Fix #1994: using * with props: true generates a param named 0
      // path中如果有*；key.name:0 的情况，path: /man-* , push({path: '/man-pathM'}) --> params.pathMatch = pathM
      params[key.name || 'pathMatch'] = val
    }
  }
  return true
}

function resolveRecordPath(path, record) {
  return resolvePath(path, record.parent ? record.parent.path : '/', true)
}
```
`match` 方法多了一个从哪里重定向过来的 `redirectedFrom` 参数。此参数通过 `_createRoute` 方法传入 `createRoute` 

## 别名
“重定向”的意思是，当用户访问 /a时，URL 将会被替换成 /b，然后匹配路由为 /b，那么“别名”又是什么呢？
::: tip 解释
/a 的别名是 /b，意味着，当用户访问 /b 时，URL 会保持为 /b，但是路由匹配则为 /a，就像用户访问 /a 一样。
:::
上面对应的路由配置为：
```js
const router = new VueRouter({
  routes: [
    { path: '/a', component: A, alias: '/b' }
  ]
})
```
“别名”的功能让你可以自由地将 UI 结构映射到任意的 URL，而不是受限于配置的嵌套路由结构。

由上面的例子我们知道，当我们访问 `/b` 的时候，其实是展示 `/a` 的路由信息。`vue-router` 是这样做的，在路由记录中新增一条记录，比如：
别名 `/b` 其创建的记录大概是这样
```js
const record = {
  path: "/b",
  matchAs: "/a",
  components: {default: undefined},
  regex: /^\/b(?:\/(?=$))?$/i,
  ……
}
```
很多属性都是 `undefined` 但是它多了一个 `matchAs` 属性，比如 `matchAs: '/a'` ，说明其是 `/a` 的别名。
我们看看是怎么添加这个 `matchAs` 的
### create-route-map
```js{21,31,42-51,54-80}
export function createRouteMap(
  routes, //传来的路由配置信息
) {
  /*…………*/
  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })
  /*…………*/
  return {
    pathList,
    pathMap,
    nameMap
  }
}
function addRouteRecord(
  pathList,
  pathMap,
  nameMap,
  route,
  parent,
  matchAs //是什么path的别名路由
) {
  //
  /*…………*/
  const record = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    name,
    parent,
    matchAs,//是什么path的别名路由
    redirect: route.redirect, // 有重定向参数
    meta: route.meta || {},
    props: //路由组件传参 有了这个功能，组件可以不再和$route耦合
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props } //非多组件的情况下，就是对默认组件的设置
  }
  // **嵌套路由**
  if (route.children) {
    /*…………*/
    route.children.forEach(child => {//这里当前的record就是下一层的parent
      //子集是谁的别名，要带上父级的path
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
  /*…………*/
  //如果有别名
  if (route.alias !== undefined) {// 别名可以通过数组传入多个别名
    const aliases = Array.isArray(route.alias) ? route.alias : [route.alias]
    for (let i = 0; i < aliases.length; ++i) {
      const alias = aliases[i]
      if (process.env.NODE_ENV !== 'production' && alias === path) {
        warn(
          false,
          `Found an alias with the same value as the path: "${path}". You have to remove that alias. It will be ignored in development.`
        )
        // skip in dev to make it work
        continue
      }
      //别名就是以 （有别名的route）创建新记录
      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(//创建路由记录
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs 谁的别名
      )
    }
  }
  /*…………*/
}
/*…………*/
```
`54-80` 行：
+ 从这段代码我们可以看出，别名是支持数组和字符串形式的，而且只会把它当做 `path` 来对待。希望别名通过 `name` 来就别想了
+ 如果别名和被别名的 `path` 一样会发生警告
+ `addRouteRecord` 最后加了个参数，被别名的`path`: `matchAs`

回到 `21` 行，把这个 `matchAs` 赋给 `record` 对象

`42-51` 行：
如果有孩子，孩子的 `matchAs` 要拼上父亲的 `path`。例子：
```js
{
  path: '/a',
  children: {
    path: 'b',
    alias: '/alias-b' // path: /alias-b 的matchAs是 /a/b 而非 b
  }
}
```





那么有了这个 `matchAs`，接下来我们看看它是如何执行的：

### create-matcher
看关键代码，其他代码省略
```js{11-13}
function _createRoute(
  record,
  location,
  redirectedFrom
) {
  if (record && record.redirect) { // 如果有重定向
    return redirect(record, redirectedFrom || location)
  }
  // 比如 record.matchAs: '/a' record.path: '/b'
  //那么配置信息应该是这样的：{path: '/a', alias: '/b'}
  if (record && record.matchAs) {
    return alias(record, location, record.matchAs)
  }
  return createRoute(record, location, redirectedFrom)
}
```
如果有是谁的别名，直接执行了 `alias` 方法

我们一点一点分析：例子：`path: '/a', alias: '/b' --> matchAs: '/a'`
+ 我们需要通过 `matchAs` 来创建路由记录，如：`aliasedRecord` 。
+ 通过 `aliasedRecord` 来创建路由对象
+ 创建的路由对象相关的 `path` 和 `fullPath` 等信息应该是 `'/b'` 的，所以还要传参当前的 `location`
```js
_createRoute(aliasedRecord, location) --> createRoute(record, location)
```
通过这样的参数创建的路由信息就是我们希望的：
```js{9,17,19}
export function createRoute(
  record,
  location
) {
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
    //一个数组，包含当前路由的所有嵌套路径片段的路由记录
    matched: record ? formatMatch(record) : []
  }
}
```
不相关代码已经省略，`path、fullPath` 大多是 `location` 带来的，而 `matched` 是 `path: '/a'` 的记录设置而来
### alias
```js{13}
function alias(
  record,
  location,
  matchAs
) {
  const aliasedPath = fillParams(matchAs, location.params, `aliased route with path "${matchAs}"`)
  const aliasedMatch = match({
    _normalized: true,
    path: aliasedPath
  })
  if (aliasedMatch) {
    const matched = aliasedMatch.matched
    const aliasedRecord = matched[matched.length - 1]
    location.params = aliasedMatch.params
    return _createRoute(aliasedRecord, location)
  }
  return _createRoute(null, location)
}
```
现在我们再看 `alias` 方法就简单多了，我们着重解析下第 `13` 行代码。`matched` 是一个数组，不懂得可以去看[嵌套路由](./nested-router.html)章节。为什么取`matched` 数组的最后一个，举个例子：
```js
matchAs: '/a/b/c'
```
通过此 `path` 匹配的 `matched`
```js
matched = [
  {……}, // /a 的record
  {……}, // /a/b 的record
  {……}  // /a/b/c 的record
]
```
