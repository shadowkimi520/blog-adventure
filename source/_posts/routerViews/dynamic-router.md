# 动态路由匹配
完整代码分支 [stage-2](https://github.com/shengrongchun/parse-vue-router)

什么是动态路由匹配呢？给个官网例子吧
```js
const User = {
  template: '<div>User</div>'
}
const router = new VueRouter({
  routes: [
    // 动态路径参数 以冒号开头
    { path: '/user/:id', component: User }
  ]
})
```
现在呢，像 `/user/foo` 和 `/user/bar` 都将映射到相同的路由。这里是[官网详解](https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html)，如果你对动态路由一无所知，请先详细查看官网介绍

### 高级匹配模式
`vue-router` 使用 `path-to-regexp` 作为路径匹配引擎，所以支持很多高级的匹配模式，例如：可选的动态路径参数、匹配零个或多个、一个或多个，甚至是自定义正则匹配。我们看一个例子
```js{5}
import Regexp from 'path-to-regexp'
const path = '/man/:id/:user'
const pathToRegexpOptions = {}//正则规则配置options
//regex.keys: [{name: id,optional: false……},{name: user,optional: false……}]
const regex = Regexp(path, [], pathToRegexpOptions)
//
const demo = '/man/123/456'
const result = demo.match(regex)
//如果有值则匹配上result: ["/man/123/456", "123", "456", index: 0, input: "/man/123/456", ……]
```
我们传入的路由配置信息会有像这样 `/man/:id/:user` 的 `path`。我们应该在创建 `record` 记录上通过传入的 `path` 增加 `regex`。供后续使用
::: tip createRouteMap 提示
`17-23` 行代码功能的[官网介绍]( https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html#%E6%8D%95%E8%8E%B7%E6%89%80%E6%9C%89%E8%B7%AF%E7%94%B1%E6%88%96-404-not-found-%E8%B7%AF%E7%94%B1)
:::
### createRouteMap
```js{17-23,58-64,67,92-108}
import Regexp from 'path-to-regexp'
import { cleanPath } from './util/path'
import { assert, warn } from './util/warn'
//
export function createRouteMap(
  routes
) {
  const pathList = [] //创建空数组
  const pathMap = Object.create(null)//创建空对象
  const nameMap = Object.create(null)//创建空对象
  //遍历 routes 把 route 相关信息放入 pathList, pathMap, nameMap
  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })
  // 如果定义了path:*,确保此path放在pathList的最后,因为正则匹配*是永远匹配上的。
  // 要在数组最后之前都没匹配上才会轮到最后的*
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }
  if (process.env.NODE_ENV === 'development') {//非生产环境，path不是以*或者/开头会警告
    // warn if routes do not include leading slashes
    const found = pathList
      // check for missing leading slash
      .filter(path => path && path.charAt(0) !== '*' && path.charAt(0) !== '/')
    if (found.length > 0) {
      const pathNames = found.map(path => `- ${path}`).join('\n')
      warn(false, `Non-nested routes must include a leading slash character. Fix the following routes: \n${pathNames}`)
    }
  }
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
) {
  const { path, name } = route
  if (process.env.NODE_ENV !== 'production') {//非生产环境警告，配置信息path是必须的
    assert(path != null, `"path" is required in a route configuration.`)
    assert(//非生产环境警告，component不能是字符串，必须是一个真实的组件
      typeof route.component !== 'string',
      `route config "component" for path: ${String(
        path || name
      )} cannot be a ` + `string id. Use an actual component instead.`
    )
  }
  //官网介绍:https://router.vuejs.org/zh/api/#router-%E6%9E%84%E5%BB%BA%E9%80%89%E9%A1%B9
  const pathToRegexpOptions = //正则配置 options
    route.pathToRegexpOptions || {}
  //标准化path
  const normalizedPath = normalizePath(path, null, pathToRegexpOptions.strict)
  if (typeof route.caseSensitive === 'boolean') {// 正则匹配规则是否大小写敏感？(默认值：false)
    pathToRegexpOptions.sensitive = route.caseSensitive
  }
  const record = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),//依赖path生成正则
    components: route.components || { default: route.component },
    name,
    meta: route.meta || {},
  }
  //去掉重复的path定义
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  //
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production') {
      warn(//非生产环境警告，配置信息name不能重复
        false,
        `Duplicate named routes definition: ` +
        `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
//path: /man/:id
//regex.keys: [{name: id,optional: false……}]
function compileRouteRegex(
  path,
  pathToRegexpOptions
) {
  const regex = Regexp(path, [], pathToRegexpOptions)
  if (process.env.NODE_ENV !== 'production') {
    const keys = Object.create(null)
    regex.keys.forEach(key => {
      warn(// 比如这样的/man/:id/:id会发出警告
        !keys[key.name],
        `Duplicate param keys in route with path: "${path}"`
      )
      keys[key.name] = true
    })
  }
  return regex
}
// 标准化path
function normalizePath(
  path,
  parent,
  strict
) {
  if (!strict) path = path.replace(/\/$/, '') // 非严格模式会去掉path最后的 /
  if (path[0] === '/') return path
  if (parent == null) return path
  return cleanPath(`${parent.path}/${path}`)
}
```
::: tip 解析
上述最重要的功能是通过 `compileRouteRegex` 方法通过传入的 `path` 参数返回对应的 `regex` 正则对象给每个 `record`
:::

### createMatcher
`createMatcher` 也有所改变，功能如下
+ 通过正则收集动态 path 参数名, 增加相对 params 功能
+ 动态路由 path 填充 params，生成最终的 path:/man/:id -->params:{id:123}-->/man/123
```js{27-29,36-42,45,55-61,115-124}
/* eslint-disable no-prototype-builtins */
import { warn } from './util/warn'
import { createRoute } from './util/route'
import { fillParams } from './util/params'
import { normalizeLocation } from './util/location'
import { createRouteMap } from './create-route-map'

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
      //通过正则来收集动态path参数名如：/a/:username/:userid paramNames: [username, userid]
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)
      //
      if (typeof location.params !== 'object') {// params 不是对象就设置成空对象
        location.params = {}
      }
      // 如果location的params里面没有如username, userid，
      // 那么看看当前路由上的params有没有，有就加进来。
      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }
      // 把params填充到path中如record.path: /a/:username/:userid; params: {username:vue,userid:router};
      // 那么最终的path为: /a/vue/router。fillParams方法请在工具方法中查看
      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location)
    } else if (location.path) {
      location.params = {} // 这句代码，让在有path的时候无视了params
      // 动态路由这里需要改造 直接的pathMap[location.path]是获取不到 record的
      // 如：pathMap:{ '/man/:id': record } location.path: /man/123
      // 我们可以轮询pathMap里的record ,通过record.regex来匹配 location.path，
      // 如果匹配上就获取到了匹配的record 当然这里是会获取最先匹配上的 
      // 即使：pathMap: { '/man/:id': record1, '/man/:user': record2 } 
      // location.path: /man/123 只会匹配到 record1
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
    return createRoute(record, location)
  }
  //
  return {
    match
  }
}

// path: /man/123/456   regex是通过 /man/:id/:user创建的正则解析对象
// regex.keys: [
//   {
//     asterisk: false
//     delimiter: "/"
//     name: "id"
//     optional: false
//     partial: false
//     pattern: "[^\/]+?"
//     prefix: "/"
//     repeat: false
//   },
//   {
//     asterisk: false
//     delimiter: "/"
//     name: "user"
//     optional: false
//     partial: false
//     pattern: "[^\/]+?"
//     prefix: "/"
//     repeat: false
//   },
// ]
// path.match(regex)：["/man/123/456", "123", "456", index: 0, input: "/man/123/456", ……]
function matchRoute(
  regex,
  path,
  params
) {
  //
  const m = path.match(regex)
  if (!m) {//没有匹配上
    return false
  } else if (!params) { //
    return true
  }
  for (let i = 1, len = m.length; i < len; ++i) {
    const key = regex.keys[i - 1]
    const val = typeof m[i] === 'string' ? decodeURIComponent(m[i]) : m[i]
    if (key) {
      // path中如果有*或者包含*；key.name就为0，
      // 如：path: /man-* , push({path: '/man-pathM'}) --> params.pathMatch = pathM
      // 官网介绍 https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html#%E6%8D%95%E8%8E%B7%E6%89%80%E6%9C%89%E8%B7%AF%E7%94%B1%E6%88%96-404-not-found-%E8%B7%AF%E7%94%B1
      params[key.name || 'pathMatch'] = val
    }
  }
  return true
}
```
[fillParams方法查看](/routerViews/util-router.html#fillparams)