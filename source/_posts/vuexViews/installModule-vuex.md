# installModule
+ store: 是 `new vuex.store` 对象也就是我们在组件中使用的 `this.$store`
+ rootState: 根模块定义的 `state`
+ path: 存放模块名的数组
+ module：遍历当前模块
+ hot：热替换标识，在之后的 `hotupdate` 方法中会详细介绍。具体可以看[官网](https://vuex.vuejs.org/zh/api/#hotupdate)
```js{3-5}
function installModule(store, rootState, path, module, hot) {
  …………
  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```
处理逻辑先隐藏掉，我们看 `3-5` 行代码可以知道，这是递归遍历模块执行 `installModule`。模块的 `forEachChild` 方法可以在 `辅助` 的 `Module` 中查看

接下来我们会一步一步展示隐藏的逻辑代码
```js
const namespace = store._modules.getNamespace(path)//获取模块对应的命名空间
// register in namespace map
if (module.namespaced) {//模块如果定义了命名空间，就在_modulesNamespaceMap中留下记录
  if (store._modulesNamespaceMap[namespace] && __DEV__) {
    console.error(`[vuex] duplicate namespace ${namespace} for the namespaced module ${path.join('/')}`)
  }
  store._modulesNamespaceMap[namespace] = module
}
```
如果模块定义了命名空间，如：
```js
模块A: {
  namespaced: true
}
```
那么此模块应该存入 `this.$store._modulesNamespaceMap` 中供后续使用。存入的 `key` 通过 `getNamespace` 方法获取
```js
//模块中定义了命名空间，在使用commit等都要加上命名空间/
//如：this.$store.commit('模块A/模块B/getName', 'helloWorld')
getNamespace(path) {
  let module = this.root
  return path.reduce((namespace, key) => {
    module = module.getChild(key)//通过模块名获取模块
    return namespace + (module.namespaced ? key + '/' : '')
  }, '')
}
```
通过判断层级模块是否定义了 `namespaced` 来字符拼接命名空间名字，例子：
```js
const 根模块 = {
  namespaced: true,//此时根模块的key: ''
  模块A: {
    namespaced: true,
    模块B: {},//此时模块B的key: '模块A/'
    模块C: { namespaced: true}//此时模块C的key: '模块A/模块C/'
  }
}
// _modulesNamespaceMap
this.$store._modulesNamespaceMap = {
  '': 根模块实体,
  '模块A/': 模块B实体,
  '模块A/模块C/': 模块C实体
}
```
接下来的逻辑隐藏代码
```js{7,8,9,21,24}
const isRoot = !path.length //是否为根模块
// set state
//hot为true说明并非初始化阶段，而是热替换新的 action 和 mutation
//如果你去了解热替换可以知道此时是不需要设置state的，沿用之前的state
//根模块的state是不需要设置的
if (!isRoot && !hot) {
  //一开始parentState就是rootState，接下来通过21行代码自我设置
  //下次遍历就会获取父级的state,如果听不懂可以结合9,21,24行代码慢慢体会
  const parentState = getNestedState(rootState, path.slice(0, -1))
  const moduleName = path[path.length - 1]
  store._withCommit(() => {
    if (__DEV__) {
      if (moduleName in parentState) {//警告：定义的state被相同模块名称重写了
        console.warn(
          `[vuex] state field "${moduleName}" was overridden by a module with the same name at "${path.join('.')}"`
        )
      }
    }
    //往根state中设置子模块的state,因为此时parentState并非响应式，所以 Vue.set只是单纯的添加新属性，这里用Vue.set不太清楚
    //这里parentState变化了，在mutation之外改变state,在严格模式下都会发出警告，为了不警告，通过_withCommit包裹
    Vue.set(parentState, moduleName, module.state)
  })
}
function getNestedState(state, path) {//通过path来获取对应模块的state
  return path.reduce((state, key) => state[key], state)
}
```
这段代码的功能是设置 `state`，希望把所有模块的 `state` 都收集到其父 `state` 里面，根模块不需要收集。`key` 通过模块名称来定义，例子：
```js
const parentState = {
  …………,
  模块A: {age,name……},//模块A的state
  模块B: {
    sex,address……,
    模块C: {……}//模块C的state
  },//模块B的state
}
```
这就是根模块 `state` 的结构，还记得我们在组件中获取某个模块的 `state` 吗？
```js
this.$store.state['模块A'].age
```

如果用户通过配置信息定义了严格模式 `strict` ，所有在 `mutation` 之外改变 `state` 都会发出警告。我们看这段代码：
```js
Vue.set(parentState, moduleName, module.state)
```
明显改变了 `state` 而且不是用 `mutation` 方法改变的，所以会发生警告。为了不警告，可以用 ` store._withCommit` 方法包裹。具体细节会在之后的严格模式章节详细介绍

接下来的隐藏逻辑代码
```js{5,11,15}
const local = module.context = makeLocalContext(store, namespace, path)
//遍历此模块的mutations,把里面定义的mutation全部收集到store._mutations
module.forEachMutation((mutation, key) => {
  const namespacedType = namespace + key
  registerMutation(store, namespacedType, mutation, local)
})
module.forEachAction((action, key) => {
  //由于在定义action的时候是可以写成对象形式{handler：function},因此action里面可以定义root
  const type = action.root ? key : namespace + key
  const handler = action.handler || action
  registerAction(store, type, handler, local)
})
module.forEachGetter((getter, key) => {
  const namespacedType = namespace + key
  registerGetter(store, namespacedType, getter, local)
})
```
模块中的三个方法 `forEachMutation forEachAction forEachGetter` 可以在 `辅助` 的 `Module` 中查看。我们看最终处理方法：

### registerMutation
```js
function registerMutation(store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler(payload) {
    handler.call(store, local.state, payload)
  })
}
```
::: tip 注释
`vuex` 会把所有模块和根模块所定义的 `mutation` 都收集起来放入 `store._mutations` 中，例如
如果 `模块A` 的 `mutations` 定义了 `age` ,根模块也定义了 `age` ,此时收集会有两种情况：

情况一：`模块A` 没有定义命名空间，那么此模块定义的 `mutation` 都会当做根模块定义处理：store._mutations['age'] = [模块A的age, 根模块的age]

情况二：`模块A` 有定义命名空间，那么此时的情况是：store._mutations['age'] = [根模块的age] store._mutations['模块A/age'] = [模块A的age]
:::
通过上面的代码可知，并没有直接将 `age` 函数存入数组中，而是中间包裹了一层
```js{2}
function wrappedMutationHandler(payload) {
  handler.call(store, local.state, payload)//handler-->age
}
```
为什么这么做？因为我们在执行我们定义的 `mutation` 需要传入参数，比如：
```js
this.$store.commit('age', '我是payload')//我们还没有讲解commit,这里只要知道commit后会执行wrappedMutationHandler
const mutations = {
  age(state, payload) {……}
}
```
通过 `handler.call` 我们可以知道，`age` 函数里面的 `this` 是 `store` , `state` 参数是 `local.state` , `payload` 就是 `commit` 传来的。接下来我们重点介绍 `local`
```js{1,5-7}
const local = module.context = makeLocalContext(store, namespace, path)
function makeLocalContext(store, namespace, path) {
  const local = {}
  Object.defineProperties(local, {
    state: {
      get: () => getNestedState(store.state, path)
    }
  })
  return local
}
```
已经把此时不相关的代码隐藏了，我们知道 `age` 第一个参数是 `state`，取的肯定是当前模块的 `state`：
`local.state --> getNestedState(store.state, path)`，通过 `path` 获取模块 `state`。
```js
function getNestedState(state, path) {//通过path来获取对应模块的state
  return path.reduce((state, key) => state[key], state)
}
```

### registerAction
为什么最终返回 `promise`，请看[官网介绍](https://vuex.vuejs.org/zh/guide/actions.html)
```js{5-10}
function registerAction(store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler(payload) {
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,//之前方法已介绍
      rootGetters: store.getters, //根模块的 getters,之后章节会介绍如何收集
      rootState: store.state //根模块的 state,之后章节会介绍如何收集
    }, payload)
    if (!isPromise(res)) {// isPromise在工具方法中查看
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
```
有了之前 `registerMutation` 的介绍，我们直接定位定义 `action` 的第一个参数，[官网介绍](https://vuex.vuejs.org/zh/api/#actions)。`local` 是如何定义 `dispatch commit getters`：
```js{6,12,20,36-40,48}
const local = module.context = makeLocalContext(store, namespace, path)
function makeLocalContext(store, namespace, path) {
  const noNamespace = namespace === '' //如果是空字符就是根模块下
  //local 是用在 mutations actions getters等里面的
  const local = {//module.context
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args
      //如果你在options 里面定义了root: true 它允许在命名空间模块里分发根的 action
      if (!options || !options.root) {
        type = namespace + type
        if (__DEV__ && !store._actions[type]) {
          console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
          return
        }
      }
      return store.dispatch(type, payload)
    },
    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args
      //如果你在options 里面定义了root: true 它允许在命名空间模块里提交根的 mutation
      if (!options || !options.root) {//type需要加上命名空间
        type = namespace + type
        if (__DEV__ && !store._mutations[type]) {//mutation没有定义此type
          console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
          return
        }
      }
      store.commit(type, payload, options)
    }
  }
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters //
        : () => makeLocalGetters(store, namespace)//获取自己模块的 getters
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })
  return local
}
//为什么type会有对象的情况，因为 Mutations Actions 支持载荷方式和对象方式进行分发
function unifyObjectStyle(type, payload, options) {
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }
  if (__DEV__) {
    assert(typeof type === 'string', `expects string as the type, but found ${typeof type}.`)
  }
  return { type, payload, options }
}
```
载荷方式和对象方式进行分发[官网介绍](https://vuex.vuejs.org/zh/guide/actions.html)

`dispatch` 和 `commit` 基本差不多，我们只讲解 `dispatch`。我们首先要知道我们如何在 `action` 中使用 `local.dispatch`
```js
const actions = {
  age({dispatch}, payload) {
    dispatch(……)
  }
}
```
一般情况下我们是 `dispatch` 自己模块的 `mutations`。收集的 `mutations` 是通过命名空间作为 `key` 的。因此 ` store.commit` 的 `type` 需要加上命名空间。当然也可以通过 `options.root` 设置，直接触发根模块的 `mutations`

```js{4}
const 模块 = {
  actions: {
    name: {//模式一
      root: true,
      handler({dispatch}) {}//此时的dispatch直接触发根模块定义的mutations
    },
    age({dispatch}) {}//模式二 此时的dispatch直接触发本模块定义的mutations
  }
}
```

`local.getters`
```js{2-6}
Object.defineProperties(local, {
  getters: {
    get: noNamespace
      ? () => store.getters //
      : () => makeLocalGetters(store, namespace)//获取自己模块的 getters
  },
  state: {
    get: () => getNestedState(store.state, path)
  }
})
```
在根模块下，local.getter === store.getters，否则获取当前模块的 `getters`

`makeLocalGetters`
```js{17}
//在store.getters中获取命名空间为namespace模块的getters
function makeLocalGetters(store, namespace) {//在
  if (!store._makeLocalGettersCache[namespace]) {
    const gettersProxy = {}
    const splitPos = namespace.length
    Object.keys(store.getters).forEach(type => {
      // skip if the target getter is not match this namespace
      if (type.slice(0, splitPos) !== namespace) return //type与namespace不匹配

      // extract local getter type
      const localType = type.slice(splitPos)

      // Add a port to the getters proxy.
      // Define as getter property because
      // we do not want to evaluate the getters in this time.
      Object.defineProperty(gettersProxy, localType, {
        get: () => store.getters[type],
        enumerable: true
      })
    })
    store._makeLocalGettersCache[namespace] = gettersProxy
  }

  return store._makeLocalGettersCache[namespace]
}
```
### registerGetter
需要把所有模块定义的 `getters` 收集到 `_wrappedGetters` 中
```js{9-14}
function registerGetter(store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    if (__DEV__) {
      console.error(`[vuex] duplicate getter key: ${type}`)
    }
    return
  }
  store._wrappedGetters[type] = function wrappedGetter(store) {
    return rawGetter(
      local.state, // local state 自己模块的state
      local.getters, // local getters 自己模块的getters
      store.state, // root state 根模块的state
      store.getters // root getters 根模块的getters
    )
  }
}
```
还记得我们怎么定义 `getters` 的吗? 具体可以查看[官网](https://vuex.vuejs.org/zh/api/#getters)
```js{8}
const store = new Vuex.Store({
  state: {
    todos: [
      { id: 1, text: '...', done: true },
    ]
  },
  getters: {
    doneTodos: (state, getters, rootState, rootGetters) => {
      return state.todos.filter(todo => todo.done)
    }
  }
})
```
