# 导航守卫
[官网介绍](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html#%E5%85%A8%E5%B1%80%E5%89%8D%E7%BD%AE%E5%AE%88%E5%8D%AB)

完整代码分支 [stage-9](https://github.com/shengrongchun/parse-vue-router)
## record 记录收集组件实例
为什么要在 `record` 记录中收集组件实例呢？因为在解析导航守卫的时候需要用到

### create-route-map.js
```js{6}
const record = {
  path: normalizedPath,
  regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
  components: route.components || { default: route.component },
  // components对应的实例，如components：{nameA:A,nameB:B}-->instances:{nameA:实例A,nameB:实例B}
  instances: {}, 
  ……
}
```
什么情况下可以获取到每个组件的实例呢？

### install.js
::: tip
还记得这里是可以获取到所有页面的实例吗
:::
```js{2-7,22,24-26}
const isDef = v => v !== undefined
const registerInstance = (vm, callVal) => {
  let i = vm.$options._parentVnode
  if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
    i(vm, callVal)
  }
}
// vue在每个实例中挂载一个属性_routerRoot
Vue.mixin({
  beforeCreate() {
    if (isDef(this.$options.router)) {//根实例
      this._routerRoot = this
      this._router = this.$options.router
      //这里需要初始化，设置当前路由route的信息，初始化方法放在了this._router(new VueRouter)上
      //因为当前路由current放在了this._router.history
      this._router.init(this) // this根实例
      Vue.util.defineReactive(this, '_route', this._router.history.current)
    } else {//其他子实例
      this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
    }
    //添加实例
    registerInstance(this, this)
  },
  destroyed() {//销毁实例
    registerInstance(this)
  }
})
```
我们定义了 `registerInstance` 方法来注册实例或者销毁实例，没有传第二个参数就是销毁实例。我们发现它执行了
```js
vm.$options._parentVnode.data
```
### view.js
看看 `view.js` 怎么定义的
```js{7-16}
 render(_, { props, children, parent, data }) {
    ……
    const matched = route.matched[depth] //record
    ……
    // attach instance registration hook
    // this will be called in the instance's injected lifecycle hooks
    data.registerRouteInstance = (vm, val) => {
      // val could be undefined for unregistration
      const current = matched.instances[name]
      if (
        (val && current !== vm) ||
        (!val && current === vm)
      ) {
        matched.instances[name] = val
      }
    }
    // also register instance in prepatch hook
      // in case the same component instance is reused across different routes
      ; (data.hook || (data.hook = {})).prepatch = (_, vnode) => {
        matched.instances[name] = vnode.componentInstance
      }

    // register instance in init hook
    // in case kept-alive component be actived when routes changed
    data.hook.init = (vnode) => {
      if (vnode.data.keepAlive &&
        vnode.componentInstance &&
        vnode.componentInstance !== matched.instances[name]
      ) {
        matched.instances[name] = vnode.componentInstance
      }
    }

 }
```
+ 如果有 `val` 值是注册，当前先要判断之前实例值和现在的实例值不同才赋值。销毁就是实例值相同的时候销毁置空。
+ `data.hook.prepatch` 这个钩子函数，在`vue`内部被调用，它是什么场景被调用呢？在`vue-router`的使用场景中，当路由相同，只是`params`不同时，路由组件实例会被重用，而不是重新`render`，此时`data.hook.prepatch`就会被调用，这个回调函数会传入两个参数，第一个参数是重用前的`vnode`，第二参数新创建的`vnode`。 在上面的源码中，在`prepatch`内部，通过第二个回调参数，更新了`matched.instances`
+ `data.hook.init`这个钩子函数，它的被调用场景是，当路由变化，组件位于`keep-alive`模式下，并且从`inactive`恢复到`active`时。 上面的源码中，在`init`函数内部，也是更新了`matched.instances`。


## 解析导航守卫

导航守卫可以很容易的控制路由跳转以及更加方便的操作路由，通过官网介绍可以发现，守卫钩子函数也分很多种：
+ 全局前置守卫
+ 全局解析守卫
+ 全局后置钩子
+ 路由独享的守卫
+ 组件内的守卫

首先我们从全局的守卫开始，在初始化的时候，定义存放全局守卫的函数，把用户定义的守卫函数存放进去：
### index.js
```js{6-9,16-30,33-39}
export default class VueRouter {
  constructor(options) {
    this.app = null //根实例
    this.apps = [] //存放多个根实例
    this.options = options
    // https://router.vuejs.org/zh/guide/advanced/navigation-guards.html
    this.beforeHooks = [] // 存放全局路由前置守卫函数数组
    this.resolveHooks = [] //存放全局路由解析守卫函数数组
    this.afterHooks = [] //存放全局路由后置钩子函数数组
    // 根据路径匹配创建 route
    this.matcher = createMatcher(options.routes || [], this)
    //创建当前路由可以弄一个公共方法，这样路由更改的时候，调用公共创建方法即可
    this.history = new HTML5History(this, options.base)
  }
  /*…………*/
  beforeEach(fn) {
    return registerHook(this.beforeHooks, fn)
  }
  beforeResolve(fn) {
    return registerHook(this.resolveHooks, fn)
  }
  afterEach(fn) {
    return registerHook(this.afterHooks, fn)
  }
  onReady(cb, errorCb) {//路由跳转不管成功还是失败，只要执行完后，用户希望执行的函数
    this.history.onReady(cb, errorCb)
  }
  onError(errorCb) {//路由跳转失败执行用户自定义的失败函数
    this.history.onError(errorCb)
  }
  /*…………*/
}
function registerHook(list, fn) {
  list.push(fn) // 把相应类型函数存放到相应类型数组中
  return () => {// 返回一个移除已注册的守卫/钩子的函数
    const i = list.indexOf(fn)
    if (i > -1) list.splice(i, 1)
  }
}
```
上述代码没什么复杂的地方，`onReady, onError` 提供了根据路由跳转后状态来执行相应自定义函数，如跳转失败，跳转完成，跳转完成报错等状态。其他的是把用户定义的导航守卫收集起来放入相应的类型数组中，导航守卫函数返回的函数执行是可以从相应类型数组中移除定义的守卫，也相当于销毁守卫。现在我们已经收集了我们需要的全局守卫函数

### base.js
```js{10-13,18,28}
export class History {
  constructor(router, base) {
    //路由实例对象
    this.router = router //$router
    //基本路径
    this.base = normalizeBase(base)
    //当前路由
    this.current = START
    //
    this.ready = false //是否执行完成ready的标识
    this.readyCbs = [] //存放ready后要执行的函数
    this.readyErrorCbs = [] //存放ready后发生错误要执行的函数
    this.errorCbs = [] //存放有错误发生后要执行的函数
    //存储一些监听事件
    this.listeners = []
  }
  /*…………*/
  onReady(cb, errorCb) {
    if (this.ready) {
      cb()
    } else {
      this.readyCbs.push(cb)
      if (errorCb) {
        this.readyErrorCbs.push(errorCb)
      }
    }
  }
  onError(errorCb) {
    this.errorCbs.push(errorCb)
  }
}
```
新增了一些存放相应类型函数的数组，还有一个路由跳转完成的标识。当然在 `onReady` 方法中，如果标识已经是完成状态，那就直接执行传入的完成函数钩子，否则都是存入相应的类型的数组中

接下来就是路由跳转了 `transitionTo`
```js
transitionTo(location, onComplete, onAbort) {
  //新匹配创建的route
  let route
  try {
    route = this.router.match(location, this.current)
    onComplete && onComplete(route)
  } catch (e) {
    onAbort && onAbort(e)
    // Exception should still be thrown
    throw e
  }
  this.updateRoute(route)
}
```
这是之前我们写的代码，其实是有一些地方处理的不妥的：
+ 第 `8` 行应该是路由跳转失败状态需要执行的函数，放在创建路由对象错误状态这里处理很不妥
+ `12` 行直接执行了路由跳转也不妥，我们应该需要判断导航守卫才能决定是否执行这段代码

更改后：
```js{8-10,14,31,39}
transitionTo(location, onComplete, onAbort) {
  //新匹配创建的route
  let route
  try {
    //可能会产生错误
    route = this.router.match(location, this.current)
  } catch (e) {
    this.errorCbs.forEach(cb => {
      cb(e)
    })
    // Exception should still be thrown
    throw e
  }
  this.confirmTransition(route, () => {//成功后
    const prev = this.current
    this.updateRoute(route)
    onComplete && onComplete(route)
    // 这里改变url地址
    this.ensureURL()
    //
    this.router.afterHooks.forEach(hook => {
      hook && hook(route, prev)
    })
    // fire ready cbs once
    if (!this.ready) {
      this.ready = true
      this.readyCbs.forEach(cb => {
        cb(route)
      })
    }
  }, (err) => {//失败后
    if (onAbort) {
      onAbort(err)
    }
    if (err && !this.ready) {
      this.ready = true
      // Initial redirection should still trigger the onReady onSuccess
      // https://github.com/vuejs/vue-router/issues/3225
      if (!isRouterError(err, NavigationFailureType.redirected)) {
        this.readyErrorCbs.forEach(cb => {
          cb(err)
        })
      } else {
        this.readyCbs.forEach(cb => {
          cb(route)
        })
      }
    }
  })
}
```
改动的比较大，我们一步一步分析：
+ 第 `8` 行是获取路由对象时候报错了，我们应该遍历错误数组执行错误回调函数
+ `confirmTransition` 目前只要知道第二个参数是路由跳转成功后执行的函数，第三个是失败后执行的函数

### 执行成功后
+ 执行 `updateRoute onComplete` 函数
+ 确保 `URL` 是对的
+ 后置导航守卫函数执行，参数是当前和之前的路由对象
+ 成功后，也是完成状态，当然要执行完成后的钩子函数

ensureURL
```js
ensureURL(push) { //确保当前路由的path和url保持一致
  if (getLocation(this.base) !== this.current.fullPath) {
    const current = cleanPath(this.base + this.current.fullPath)
    push ? pushState(current) : replaceState(current)
  }
}
```
### 执行失败后
+ 执行路由完成钩子函数
+ 路由跳转如果是重定向报错了，不应该算当前路由跳转错误，可看英文注释

::: tip 提示
`39` 行代码是判断是否为重定向错误类型，这里先不解析，下面章节会详细介绍
:::
### confirmTransition
首先我们判断当前路由和将要跳转的路由如果相同的话，就不应该再跳转。其实这是一个路由类型的错，是相同路由多次跳转的错
```js{21-30}
confirmTransition(route, onComplete, onAbort) {
  const current = this.current
  const abort = err => {
    if (!isRouterError(err) && isError(err)) {//是个错误但不是路由错误
      if (this.errorCbs.length) {
        this.errorCbs.forEach(cb => {
          cb(err)
        })
      } else {
        warn(false, 'uncaught error during route navigation:')
        console.error(err)
      }
    }
    onAbort && onAbort(err)
  }
  //怎么比较才算是相同的路由和视图：routeA：[view,subView,subSubView……]
  // 在 isSameRoute(route, current)的提前下
  // routeB的length和routeA一样，并且最后一个view也一样
  const lastRouteIndex = route.matched.length - 1
  const lastCurrentIndex = current.matched.length - 1
  if (
    isSameRoute(route, current) &&
    // in the case the route map has been dynamically appended to
    lastRouteIndex === lastCurrentIndex &&
    route.matched[lastRouteIndex] === current.matched[lastCurrentIndex]
  ) {
    this.ensureURL()//确保当前路由的path和url保持一致
    //执行未跳转成功操作，执行相关类型的函数
    return abort(createNavigationDuplicatedError(current, route))
  }
}
```
### 怎么才算相同路由
+ 嵌套页面层级相同，最后的页面相同
+ 浏览器 `URL` 的 `path, hash, query` 等相同
相同路由跳转，我们执行 `abort` 方法，并创建一个路由错误对象当作参数传入

### createNavigationDuplicatedError
```js
export const NavigationFailureType = {
  redirected: 1, //重定向错误类型码
  aborted: 2,
  cancelled: 3,
  duplicated: 4 //跳转相同路由错误码 （冗余导航）
}
//冗余导航错误
export function createNavigationDuplicatedError(from, to) {
  return createRouterError(
    from,
    to,
    NavigationFailureType.duplicated,
    `Avoided redundant navigation to current location: "${from.fullPath}".`
  )
}
```
### createRouterError
```js
//创建一个路由类型错误
function createRouterError(from, to, type, message) {
  const error = new Error(message)
  error._isRouter = true
  error.from = from
  error.to = to
  error.type = type
  return error
}
```
现在是不是已经知道路由错误是怎么回事了，就是人为自定义的错误罢了。现在再看 `isRouterError` 是否更好理解
```js
export function isError(err) {//是一个错误
  return Object.prototype.toString.call(err).indexOf('Error') > -1
}
export function isRouterError(err, errorType) {//是一个router错误
  return isError(err) && err._isRouter && (errorType == null || err.type === errorType)
}
```
### 如果不是相同路由呢？
我们之前已经收集了全局守卫，但是影响路由跳转的不仅仅是全局守卫。请看官网的[介绍](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html#%E5%AE%8C%E6%95%B4%E7%9A%84%E5%AF%BC%E8%88%AA%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B)
。从完成的导航解析流程来看需要去收集 `2-8` 的钩子函数，从当前路由跳转到新路由，页面会有什么变化呢
+ 有些页面会销毁：beforeRouteLeave
+ 有些页面会更新：beforeRouteUpdate 
+ 有些页面会被激活：beforeRouteEnter
```js{12-27}
/*…………*/
if (
  isSameRoute(route, current) &&
  // in the case the route map has been dynamically appended to
  lastRouteIndex === lastCurrentIndex &&
  route.matched[lastRouteIndex] === current.matched[lastCurrentIndex]
) {
  this.ensureURL()//确保当前路由的path和url保持一致
  //执行未跳转成功操作，执行相关类型的函数
  return abort(createNavigationDuplicatedError(current, route))
}
const { updated, deactivated, activated } = resolveQueue(
  this.current.matched,
  route.matched
)
const queue = [].concat(
  // in-component leave guards
  extractLeaveGuards(deactivated), // 放入beforeRouteLeave 销毁页面
  // global before hooks
  this.router.beforeHooks, // 放入beforeEach 全局前置钩子
  // in-component update hooks
  extractUpdateHooks(updated), // 放入beforeRouteUpdate 更新页面
  // in-config enter guards
  activated.map(m => m.beforeEnter), // 这里放入调路由配置里定义的 beforeEnter
  // async components
  resolveAsyncComponents(activated) //解析异步路由组件
)
```
把相关钩子函数根据官网定义的顺序存入 `queue` 数组中，`resolveQueue` 方法可以查看工具方法。它的作用是分析当前路由和跳转后的路由 `matched` 对比得出 `updated(更新), deactivated(销毁), activated(激活)` 的 `record`

解析 `extractLeaveGuards` `extractUpdateHooks`
```js{28,29,31,32}
function extractGuards(
  records,
  name,
  bind,
  reverse //离开钩子应该是子到父的执行顺序，所以要倒序，而更新钩子是父到子的顺序
) {
  // def：组件component; instance: 组件实例this; match: record; key: 如default
  const guards = flatMapComponents(records, (def, instance, match, key) => {
    const guard = extractGuard(def, name) // 匹配组件是否有如beforeRouteLeave等这样的函数，有返回这样的函数
    if (guard) {
      return Array.isArray(guard) //如果是数组相当于在页面定义多个相同钩子
        ? guard.map(guard => bind(guard, instance, match, key))
        : bind(guard, instance, match, key)
    }
  })
  return flatten(reverse ? guards.reverse() : guards)
}
function extractGuard( // 返回如beforeRouteLeave等这样的函数
  def,
  key
) {
  if (typeof def !== 'function') {
    // extend now so that global mixins are applied.
    def = _Vue.extend(def)
  }
  return def.options[key]
}
function extractLeaveGuards(deactivated) {//离开 true参数是离开beforeRouteLeave是从里往外离开，最里面，数组上也就是最后面最先执行
  return extractGuards(deactivated, 'beforeRouteLeave', bindGuard, true)
}
function extractUpdateHooks(updated) {//更新
  return extractGuards(updated, 'beforeRouteUpdate', bindGuard)
}
function bindGuard(guard, instance) {
  if (instance) {
    return function boundRouteGuard() {//返回执行beforeRouteLeave函数的函数，把这些函数放入queue中等待执行
      return guard.apply(instance, arguments)
    }
  }
}
```
`extractLeaveGuards` `extractUpdateHooks` 内部都执行了 `extractGuards` 方法。所以我们直接定位此方法，至于第四个参数的不同可以看代码注释。`flatMapComponents` 直接查看工具方法。得到的 `guards` 是钩子函数数组。

+ 第 `9` 行代码是收集页面的钩子函数，当然也有可能是数组类型，因为可能会在页面上定义了多个相同钩子函数
+ `12,13` 行用到的 `bind(bindGuard)` 是对钩子函数的处理，确保钩子函数的 `this` 是当前实例。最终 `guards` 数组收集了相关钩子函数，如像这样的
```js
const guards = [
  function boundRouteGuard() {// A组件
    return guard.apply(instance, arguments)
  },
  function boundRouteGuard() {// B组件
    return guard.apply(instance, arguments)
  },
  ……
]
```
我们再次看看 `queue`数组
```js{9,11}
const queue = [].concat(
  // in-component leave guards
  extractLeaveGuards(deactivated), // 放入beforeRouteLeave 销毁页面
  // global before hooks
  this.router.beforeHooks, // 放入beforeEach 全局前置钩子
  // in-component update hooks
  extractUpdateHooks(updated), // 放入beforeRouteUpdate 更新页面
  // in-config enter guards
  activated.map(m => m.beforeEnter), // m是record;这里放入调路由配置里定义的 beforeEnter
  // async components
  resolveAsyncComponents(activated) //解析异步路由组件
)
```
第 `9` 行代码需要在 `record` 中定义 `beforeEnter` ，那我们回到定义 `record` 的代码中
```js{9}
const record = {
  path: normalizedPath,
  regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
  components: route.components || { default: route.component },
  name,
  parent,
  matchAs,//是什么path的别名路由
  redirect: route.redirect, // 有重定向参数
  beforeEnter: route.beforeEnter, //路由配置里定义的 beforeEnter
  meta: route.meta || {},
  props: //路由组件传参 有了这个功能，组件可以不再和$route耦合
    route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props } //非多组件的情况下，就是对默认组件的设置
}
```
加上之后，我们再次回到 `queue` 去解析异步路由组件
::: tip 提问？
什么是异步组件？为什么在这里需要解析异步组件，
:::
异步组件是什么？如我们在路由配置中是这样定义的
```js{4}
{
  routes: [{
    path: '/bar',
    component: () => import('./Bar')
  }]
}
```
那为什么要在这里解析异步组件呢？因为如果我们路由导航到此组件，也就是激活状态，恰好激活的组件是异步组件。那我们就应该解析它，否则之后的组件 `render` 没法进行
### 解析异步组件 
#### resolveAsyncComponents(activated)
```js
export function resolveAsyncComponents(matched) {
  return (to, from, next) => {
    let hasAsync = false
    let pending = 0
    let error = null

    flatMapComponents(matched, (def, _, match, key) => {
      // if it's a function and doesn't have cid attached,
      // assume it's an async component resolve function.
      // we are not using Vue's default async resolving mechanism because
      // we want to halt the navigation until the incoming component has been
      // resolved.
      if (typeof def === 'function' && def.cid === undefined) {//异步组件
        hasAsync = true
        pending++

        const resolve = once(resolvedDef => {
          if (isESModule(resolvedDef)) {
            resolvedDef = resolvedDef.default
          }
          // save resolved on async factory in case it's used elsewhere
          def.resolved = typeof resolvedDef === 'function'
            ? resolvedDef
            : _Vue.extend(resolvedDef)
          match.components[key] = resolvedDef
          pending--
          if (pending <= 0) {
            next()
          }
        })

        const reject = once(reason => {
          const msg = `Failed to resolve async component ${key}: ${reason}`
          process.env.NODE_ENV !== 'production' && warn(false, msg)
          if (!error) {
            error = isError(reason)
              ? reason
              : new Error(msg)
            next(error)
          }
        })

        let res
        try {
          res = def(resolve, reject)
        } catch (e) {
          reject(e)
        }
        if (res) {
          if (typeof res.then === 'function') {
            res.then(resolve, reject)
          } else {
            // new syntax in Vue 2.3
            const comp = res.component
            if (comp && typeof comp.then === 'function') {
              comp.then(resolve, reject)
            }
          }
        }
      }
    })

    if (!hasAsync) next()
  }
}
```
首先它也是返回一个函数，放入 `queue` 中，当 `queue` 遍历执行这个函数时就是解析异步组件的时候。标准的 `to、from、next` 参数，它的内部实现很简单，利用了 `flatMapComponents` 方法从 `matched` 中获取到每个组件的定义，判断如果是异步组件，则执行异步组件加载逻辑，加载成功后会执行 `match.components[key] = resolvedDef` 把解析好的异步组件放到对应的 `components` 上，并且执行 `next` 函数。接着 `queue` 继续往下解析
::: tip 提示
先去工具方法中认真看 `runQueue` 方法的功能。遍历 `queue` 执行 `iterator` 方法
:::
```js{4,11,15,18,30,32}
//queue: [undefined,……,function(to,from,next),……]
this.pending = route // to 路由
// hook: queue[index] next: ()=> {step(index+1)}
const iterator = (hook, next) => {
  if (this.pending !== route) {//不相等取消路由跳转
    return abort(createNavigationCancelledError(current, route))
  }
  try {
    //hook是导航守卫函数，三个参数：（to,from,next）
    hook(route, current, (to) => {//第三个参数相当于我们使用钩子函数时候的next函数
      if (to === false) {
        // next(false) -> abort navigation, ensure current URL
        this.ensureURL(true)
        abort(createNavigationAbortedError(current, route))
      } else if (isError(to)) {
        this.ensureURL(true)
        abort(to)
      } else if (
        typeof to === 'string' ||
        (typeof to === 'object' &&
          (typeof to.path === 'string' || typeof to.name === 'string'))
      ) {
        // next('/') or next({ path: '/' }) -> redirect 重定向
        abort(createNavigationRedirectedError(current, route))
        if (typeof to === 'object' && to.replace) {
          this.replace(to)
        } else {
          this.push(to)
        }
      } else {
        // confirm transition and pass on the value 比如 next()
        next(to)
      }
    })
  } catch (e) {
    abort(e)
  }
}
//
runQueue(queue, iterator, () => {//这是cb,如果执行到这里，说明queue中定义的钩子函数都成功next了

})
```
`11,15,18,30` 行都是对用户传入 `next` 参数的判断做出相应的处理，如果执行到了 `32` 行代码，说明可以执行下一个钩子了，继续执行 `queue[index]` --> `iterator`。如果一些顺序，最终会执行 `40` 行代码的第三个参数 `cb`
::: tip
现在 `queue` 里面装入的钩子函数都执行了
:::
1. ~~导航被触发。~~
2. ~~在失活的组件里调用 beforeRouteLeave 守卫。~~
3. ~~调用全局的 beforeEach 守卫。~~
4. ~~在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。~~
5. ~~在路由配置里调用 beforeEnter。~~
6. ~~解析异步路由组件。~~
7. 在被激活的组件里调用 beforeRouteEnter。
8. 调用全局的 beforeResolve 守卫 (2.5+)。
9. 导航被确认。
10. ~~调用全局的 afterEach 钩子。~~
11. 触发 DOM 更新。
12. 调用 beforeRouteEnter 守卫中传给 next 的回调函数，创建好的组件实例会作为回调函数的参数传入。
接下来先解析 `7,8,9,11`
```js{6,7}
runQueue(queue, iterator, () => {//cb
  const enterGuards = extractEnterGuards(activated) //在被激活的组件里调用 beforeRouteEnters
  const queue = enterGuards.concat(this.router.resolveHooks)//beforeRouteEnters+全局beforeResolve
  runQueue(queue, iterator, () => {
    if (this.pending !== route) {
      return abort(createNavigationCancelledError(current, route))
    }
    this.pending = null
    onComplete(route)//经历九九81难最终成功导航后执行了这里
  })
})
```
此时的 `queue` 已经是 `beforeRouteEnters` + `beforeResolve`，然后又去执行了 `runQueue`。
### extractEnterGuards
```js{18-20}
function extractEnterGuards(
  activated,
) {
  return extractGuards(//获取守卫函数方法，之前有解析过
    activated,
    'beforeRouteEnter',
    (guard, _, match, key) => {
      return bindEnterGuard(guard, match, key)
    }
  )
}
function bindEnterGuard(
  guard,
  match,
  key,
) {
  return function routeEnterGuard(to, from, next) {
    return guard(to, from, cb => {
      next(cb)
    })
  }
}
```
最终返回的 `beforeRouteEnter` 函数是 `18` 行的 `guard` 函数，而放入 `queue` 中的函数是 `17` 行
比如：
```js
beforeRouteEnter(to, from, next) {
  next(123)
}
```
这样的定义等价于：
```js
guard(to, from, 123 => {
  next(123)
})
```
这里的 `next` 是 `queue` 遍历执行 `17` 行函数注入的参数，此参数 `next` 会根据传入的参数判断，之前有过解析。那假如 `123` 是函数，我们希望这个函数的第一个参数是实例怎么办呢？
```js{2,3,4,14-16}
runQueue(queue, iterator, () => {//cb
  const postEnterCbs = [] //装 poll 函数，poll中获取实例执行next( (vm)=> {} )中的 (vm)=> {}
  const isValid = () => this.current === route // 当前路由和跳转成功后的路由一样
  const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid) //在被激活的组件里调用 beforeRouteEnters
  const queue = enterGuards.concat(this.router.resolveHooks)//beforeRouteEnters+全局beforeResolve
  runQueue(queue, iterator, () => {
    if (this.pending !== route) {
      return abort(createNavigationCancelledError(current, route))
    }
    this.pending = null
    onComplete(route)//经历九九81难最终成功导航后执行了这里
    if (this.router.app) {
      this.router.app.$nextTick(() => {//$nextTick下通常是可以获取到实例的
        postEnterCbs.forEach(cb => {//执行用户在beforeRouteEnters自定义的回调函数
          cb() //-->poll(cb, match.instances, key, isValid)
        })
      })
    }
  })
})
```
创建一个数组 `postEnterCbs ` 装入 `poll` 函数。一个标识 `isValid` 此时此刻，当前路由和跳转路由是否相同
```js{3,4,10,18,19,23-27}
function extractEnterGuards(
  activated,
  cbs,
  isValid
) {
  return extractGuards(
    activated,
    'beforeRouteEnter',
    (guard, _, match, key) => {
      return bindEnterGuard(guard, match, key, cbs, isValid)
    }
  )
}
function bindEnterGuard(
  guard,
  match,
  key,
  cbs,
  isValid
) {
  return function routeEnterGuard(to, from, next) {
    return guard(to, from, cb => {
      if (typeof cb === 'function') {
        cbs.push(() => {
          poll(cb, match.instances, key, isValid)
        })
      }
      next(cb)
    })
  }
}
……
```
`23` 行判断用户传入的参数是否是一个函数，如果是就把 `()=> {poll()}` 放入 `postEnterCbs` 然后执行 `()=> {poll(cb)}`
```js
beforeRouterEnter(to,from,next) {
  const cb = (vm)=> {
    ……
  }
  next(cb) 
}
```
直接看 `poll`
```js{11}
function poll(
  cb, // somehow flow cannot infer this is a function
  instances,
  key,
  isValid
) {
  if (
    instances[key] &&
    !instances[key]._isBeingDestroyed // 如果有实例，并且不是正在销毁的实例执行cb
  ) {
    cb(instances[key])
  } else if (isValid()) {
    setTimeout(() => {
      poll(cb, instances, key, isValid)
    }, 16)
  }
}
```
`11` 行代码让我们定义的函数参数有实例的原因。如果没有实例并且跳转路由和当前路由相同，考虑到一些路由组件被套 `transition` 組件在一些缓动模式下不一定能拿到实例，所以用一个轮询方法不断去判断，直到能获取到组件实例

1. ~~导航被触发。~~
2. ~~在失活的组件里调用 beforeRouteLeave 守卫。~~
3. ~~调用全局的 beforeEach 守卫。~~
4. ~~在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。~~
5. ~~在路由配置里调用 beforeEnter。~~
6. ~~解析异步路由组件。~~
7. ~~在被激活的组件里调用 beforeRouteEnter。~~
8. ~~调用全局的 beforeResolve 守卫 (2.5+)。~~
9. ~~导航被确认。~~
10. ~~调用全局的 afterEach 钩子。~~
11. ~~触发 DOM 更新。~~
12. ~~调用 beforeRouteEnter 守卫中传给 next 的回调函数，创建好的组件实例会作为回调函数的参数传入。~~
