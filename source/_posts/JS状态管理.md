---
title: JS状态管理
date: 2020-08-10 12:20:56
tags: 状态管理
category: 状态管理
---

# JS 状态管理

在前端进入框架时代时代后，开发也逐渐地模块化，组件化。其中组件之间的通信也就变得尤为的重要。根据数据的变化更新`UI`，数据的状态管理在复杂的应用中，尤其是数据的共享性较高的时候。

![](E:\coding\myWebSite\source\_posts\JS状态管理\state.jpg)

![](E:\coding\myWebSite\source\_posts\JS状态管理\redux.jpg)

上面两张图片就很好的解释了有无状态管理在数据共享，改变是的便捷性。

状态管理的框架也是有很多的，下面主要讲的就是`redux`

## `redux`

2014年 Facebook 提出了 [Flux](http://www.ruanyifeng.com/blog/2016/01/flux.html) 架构的概念，引发了很多的实现。2015年，[Redux](https://github.com/reactjs/redux) 出现，将 Flux 与函数式编程结合一起，很短时间内就成为了最热门的前端架构

对于`redux`的使用，这里已`react-redux`为例

```javascript
//reducers/index.js
//首先定义一个自己的store，并将自己定义的store与redux关联起来
import { combineReducers } from 'redux'
import counter from "./list"

export default combineReducers({
    counter
})

const INITIAL_STATE = {
    num : 0
}

export default function counter(state = INITIAL_STATE , action){
    switch(action.type){
        case ADD:
            return {
                ...state,
                num++
            }
        default:
            return state
    }
}



//store/index.js
//然后我们将上面自己创建的store注册进去，并且可以注册中间件
import {
    createStore,
    applyMiddleware,
    compose
} from "redux"
import thunkMiddleware  from "redux-thunk"

import rootReducer from "../reducers"

const composeEnhancers =
  typeof window === 'object' &&
  window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ ?   
    window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__({
    }) : compose


const middlewares = [
    thunkMiddleware
  ]

  const enhancer = composeEnhancers(
    applyMiddleware(...middlewares),
    // other store enhancers if any
  )
  
  export default function configStore () {
    const store = createStore(rootReducer, enhancer)
    return store
  }

//在组件上注册
import {connect} from "react-redux"
....
const getcounter= state=>{
  return {
    counter : state.counter
  }
}
export default  connect(getcounter , null)(App)

```

总结`redux`的工作流程

- 将状态统一放在一个state中，由`store`来管理这个`state`。
- 这个`store`按照`reducer`的`“shape”`（形状）创建。
- `reducer`的作用是接收到`action`后，输出一个新的状态，对应地更新`store`上的状态。
- 根据`redux`的原则指导，外部改变`state`的最佳方式是通过调用`store`的`dispatch`方法，触发一个`action`，这个`action`被对应的`reducer`处理，完成`state`更新。
- 可以通过`subscribe`在`store`上添加一个监听函数。每当调用`dispatch`方法时，会执行所有的监听函数。
- 可以添加中间件（中间件是干什么的我们后面讲）处理副作用

### `combineReducers`

`combineReducers`可以将多个`Reducers`合并到同一个。

```javascript
//combineReducers({
//    counter
//})
export default function combineReducers(reducers: ReducersMapObject) {
  const reducerKeys = Object.keys(reducers) //拿到参数的所有keys
  const finalReducers: ReducersMapObject = {}
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]

    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }

    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)

  // This is used to make sure we don't warn about the same
  // keys multiple times.
  let unexpectedKeyCache: { [key: string]: true }
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }

  let shapeAssertionError: Error
  try {
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }

  return function combination(  //执行更新操作,相当与执行了reduer函数，更新了状态
    state: StateFromReducersMapObject<typeof reducers> = {},
    action: AnyAction
  ) {
    if (shapeAssertionError) {
      throw shapeAssertionError
    }

    if (process.env.NODE_ENV !== 'production') {
      const warningMessage = getUnexpectedStateShapeWarningMessage(
        state,
        finalReducers,
        action,
        unexpectedKeyCache
      )
      if (warningMessage) {
        warning(warningMessage)
      }
    }

    let hasChanged = false
    const nextState: StateFromReducersMapObject<typeof reducers> = {}
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    hasChanged =
      hasChanged || finalReducerKeys.length !== Object.keys(state).length
    return hasChanged ? nextState : state
  }
}
```

### `createStore`

创建，返回新的完整的`store`，并且初始化定义方法,提供相应的函数方法,以便调用

```javascript
export default function createStore<
  S,
  A extends Action,
  Ext = {},
  StateExt = never
>(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S> | StoreEnhancer<Ext, StateExt>,
  enhancer?: StoreEnhancer<Ext, StateExt>
): Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext {
  if (...){} //前面都是一些类型判断 
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    //如果preloadedState...  会被当作enhancer处理
    enhancer = preloadedState as StoreEnhancer<Ext, StateExt>
    preloadedState = undefined
  }  
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(
      reducer,
      preloadedState as PreloadedState<S>
    ) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
  }

  let currentReducer = reducer
  let currentState = preloadedState as S
  let currentListeners: (() => void)[] | null = []
  let nextListeners = currentListeners
  let isDispatching = false

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  function getState(): S {
      ...
  }

  function subscribe(listener: () => void) {
      ...
  }
      
  function dispatch(action: A) {
      ...
  }

  function replaceReducer<NewState, NewActions extends A>(
      ...
  }

  function observable() {
      ...
  }

  dispatch({ type: ActionTypes.INIT } as A)

  const store = ({
    dispatch: dispatch as Dispatch<A>,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  } as unknown) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext
  return store
}
```

#### `dispatch`

执行相应的`action`函数，以便后面调用函数改变`store`中`state`的状态

```javascript
function dispatch(action: A) {
    //if  .....
    try {
      isDispatching = true 
      // currentReducer = reducer    currentState = preloadedState as S
      currentState = currentReducer(currentState, action) //新的state
    } finally {
      isDispatching = false
    }

    const listeners = (currentListeners = nextListeners) //[]
    //执行监听列表
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }
```

#### `getState`

获取`state`

```javascript
  function getState(): S {
    //if ...
    return currentState as S
  }
```

#### `subscribe`

注册相应的`action`到监听表

```javascript
  function subscribe(listener: () => void) {

    let isSubscribed = true

    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() { //返回取消注册函数的函数
      if (!isSubscribed) {
        return
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
      currentListeners = null
    }
  }

function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }
```

当`store`创建之后，调用

```javascript
dispatch({ type: ActionTypes.INIT } as A)  //执行type 为 INIT的action，使store回到最初的状态
```

就是通过`dispatch`去执行注册的`reducer`执行相应的变化，然后返回出来不同的状态

### `applyMiddleware`

在`redux`中中间件模型，与`koa`相似.主要是对于`dispatch`的处理

```javascript
export default function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any> {
  return (createStore: StoreEnhancerStoreCreator) => <S, A extends AnyAction>(
    reducer: Reducer<S, A>,
    preloadedState?: PreloadedState<S>
  ) => {
    const store = createStore(reducer, preloadedState)
    let dispatch: Dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
      //遍历每个中间件，对于每个中间件添加相应的API，chain = [middleware1 ,middleware2 , middleware3]
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    //middleware3(middleware2(middleware1))
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```



## `VUEX`

首先在`VUEX`中要明白几个关键的东西

- `Mutation`

  - 官方解释 : 更改 `Vuex` 的 `store` 中的状态的唯一方法是提交 `mutation`。`Vuex` 中的 `mutation` 非常类似于事件：每个 `mutation` 都有一个字符串的 **事件类型 (type)** 和 一个 **回调函数 (handler)**。这个回调函数就是我们实际进行状态更改的地方，并且它会接受 state 作为第一个参数：

  - ```javascript
    const store = new Vuex.Store({
      state: {
        count: 1
      },
      mutations: {
        increment (state) {
          // 变更状态
          state.count++
        }
      }
    })
    ```

- `Action `

  - `Action` 类似于 `mutation`，不同在于

    - `Action`提交的是 `mutation`，而不是直接变更状态。

    - `Action` 可以包含任意异步操作

    - ```javascript
      const store = new Vuex.Store({
        state: {
          count: 0
        },
        mutations: {
          increment (state) {
            state.count++
          }
        },
        actions: {
          increment (context) {
            context.commit('increment')
          }
        }
      })
      ```

    - `Action` 函数接受一个与 `store` 实例具有相同方法和属性的 `context` 对象，因此你可以调用 `context.commit` 提交一个 `mutation`

- `getters`

  - 可以认为是 `store` 的计算属性 , 就像计算属性一样，`getter` 的返回值会根据它的依赖被缓存起来，且只有当它的依赖值发生了改变才会被重新计算

  - ```javascript
    const store = new Vuex.Store({
      state: {
        todos: [
          { id: 1, text: '...', done: true },
          { id: 2, text: '...', done: false }
        ]
      },
      getters: {
        doneTodos: state => {
          return state.todos.filter(todo => todo.done)
        }
      }
    })
    ```

###  基本的用法:

```javascript
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})


store.commit('increment')
console.log(store.state.count) // -> 1
```

### 源码

#### `store`

首先是对于`store`的初始化

```javascript
export class Store {
  constructor (options = {}) {
    //检查环境
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }

    const {
      plugins = [],
      strict = false
    } = options
    // store internal state
    //Object.create(null)代表是真的null，没有原型链
    this._committing = false                      //用于标识是否处于mutation处理中
    this._actions = Object.create(null)           //用来保存所有的action
    this._actionSubscribers = []                  //保存订阅action的回调
    this._mutations = Object.create(null)         //用于保存所有的 mutation，里面会先包装一次
    this._wrappedGetters = Object.create(null)    //封装的getters，通过闭包替换真正getter的参数
    this._modules = new ModuleCollection(options)  //用于保存一棵module树
    this._modulesNamespaceMap = Object.create(null) //用于保存 namespaced 的模块
    this._subscribers = []
    this._watcherVM = new Vue()                    //主要用于公共api watch方法，监听参数的变化
    this._makeLocalGettersCache = Object.create(null)

    // bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    this.strict = strict

    const state = this._modules.root.state //options
    //安装，注册每个moudle , 注册相应的dispatch , getters , state
    installModule(this, state, [], this._modules.root)
      
    resetStoreVM(this, state)

    // apply plugins
    plugins.forEach(plugin => plugin(this))

    const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
    if (useDevtools) {
      devtoolPlugin(this)
    }
  }

  get state () {
    return this._vm._data.$$state
  }

  set state (v) {
    if (__DEV__) {
      assert(false, `use store.replaceState() to explicit replace store state.`)
    }
  }

  commit (_type, _payload, _options) {
      ....
  }

  dispatch (_type, _payload) {
      ...
  }

  subscribe (fn, options) {
      ...
  }

  subscribeAction (fn, options) {
      ...
  }

  watch (getter, cb, options) {
      ...
  }

  replaceState (state) {
      ...
  }

  registerModule (path, rawModule, options = {}) {
      ...
  }

  unregisterModule (path) {
      ...
  }

  hasModule (path) {
  }

  hotUpdate (newOptions) {
  }

  _withCommit (fn) {
  }
      
}
```

##### `this._modules`

在`store`中的属性`this._modules`原型, 用来存储整棵`store`树的存在,首先会将`root`注册在`this.moudle`上，然后在对`root`的子`moudle`注册在父元素上面，形成一棵树

关于`_modules`的存在，官方示例

```javascript
const moduleA = {
  state: () => ({ ... }),
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: () => ({ ... }),
  mutations: { ... },
  actions: { ... }
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})

store.state.a // -> moduleA 的状态
store.state.b // -> moduleB 的状态
```

```javascript
export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  get (path) {
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)  //相当与递归取值
  }

  getNamespace (path) {
  }
  //热更新  
  update (rawRootModule) {
  }

  //注册模块  
  register (path, rawModule, runtime = true) {
    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1) //删除最后一个元素.slice原始数组不会改变)
      parent.addChild(path[path.length - 1], newModule) //这边应该也是对子moudle进行添加，并标记相应的path
    }
    /**
    addChild (key, module) {
      this._children[key] = module
    }
    **/

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        //[].concat 同样也是返回新的数组，不会对原来的数组产生改变  
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  unregister (path) {
  }

  isRegistered (path) {
  }
}
```

##### `installModule`

模块的安装,下面就是`store`的核心代码了

```javascript
//const state = this._modules.root.state
//installModule(this, state //options, [], this._modules.root)

function installModule (store, rootState, path, module, hot) {
  //判断是否为根节点
  const isRoot = !path.length   //true
  //获取命名空间
  const namespace = store._modules.getNamespace(path)
  /**
  获取 namespace，root 没有 namespace
  对于 modules: {a: moduleA} 来说
  namespace = 'a/'
  getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }
  **/

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state根据模块添加
 // state: { xxx: 1, a: {...}, b: {...} }
 //如果这个时候是子模块，并且并不在热更新状态，那么他就是一个新的子模块 ， 需要添加
 //首先获取它的父亲节点，然后添加    
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
/**
** function getNestedState (state, path) {
**   return path.reduce((state, key) => state[key], state)
**} 递归取到的state[path[0]][path1] ....
**/    
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
  //获取moudle的上下文  
  const local = module.context = makeLocalContext(store, namespace, path)
  // 使用模块的方法挨个为 mutation, action, getters, child 注册
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```

##### `makeLocalContext`

用来生成每个`moudle`的上下文的

```javascript
function makeLocalContext (store, namespace, path) {
  //判断是否为根节点  
  const noNamespace = namespace === ''

  const local = {
    //注册dispatch方法  
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (__DEV__ && !store._actions[type]) {
          return
        }
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (__DEV__ && !store._mutations[type]) {
          return
        }
      }

      store.commit(type, payload, options)
    }
  }

   // getters 和 state 必须实时获取，因为他们会被vm update修改
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}
```

##### `unifyObjectStyle`

用于统一对象风格

```javascript
function unifyObjectStyle (type, payload, options) {
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }

  return { type, payload, options }
}
```

##### `makeLocalGetters`

为每个新的`store`注册新的`getters` , 并标记在`store._makeLocalGettersCache`中

```javascript
function makeLocalGetters (store, namespace) {
  if (!store._makeLocalGettersCache[namespace]) {
    const gettersProxy = {}
    const splitPos = namespace.length
    Object.keys(store.getters).forEach(type => {
      // skip if the target getter is not match this namespace
      if (type.slice(0, splitPos) !== namespace) return

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

##### `mutation / action / getter`的注册

- `mutation`

  -  ```javascript
    module.forEachMutation((mutation, key) => {
        const namespacedType = namespace + key
        registerMutation(store, namespacedType, mutation, local)
      })
     ```

  - `registerMutation`

  - ```javascript
    function registerMutation (store, type, handler, local) {
      const entry = store._mutations[type] || (store._mutations[type] = [])
      entry.push(function wrappedMutationHandler (payload) {
        handler.call(store, local.state, payload)
      })
    }
    ```

  - 将`mutation`存在`store._mutations`中，上面可以看出来，`mutation`是可以重复的，且不会被覆盖

-  `action`

  -  ```javascript
    function registerAction (store, type, handler, local) {
      const entry = store._actions[type] || (store._actions[type] = [])
      entry.push(function wrappedActionHandler (payload) {
        let res = handler.call(store, {
          dispatch: local.dispatch,
          commit: local.commit,
          getters: local.getters,
          state: local.state,
          rootGetters: store.getters,
          rootState: store.state
        }, payload)
        if (!isPromise(res)) {
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

  - 同样的，将`action`注册在`store._action`上，在将注册的`action  promise`话， 将`action`链式话，可以进行链式调用

-  `getter`

  - ```javascript
    function registerGetter (store, type, rawGetter, local) {
      if (store._wrappedGetters[type]) {
        if (__DEV__) {
          console.error(`[vuex] duplicate getter key: ${type}`)
        }
        return
      }
      store._wrappedGetters[type] = function wrappedGetter (store) {
        return rawGetter(
          local.state, // local state
          local.getters, // local getters
          store.state, // root state
          store.getters // root getters
        )
      }
    }
    ```

  - `getter`就是一样存在`store._wrappedGetters`里面

##### `resetStoreVM`

核心逻辑是将`_wrappedGetters`挂载到一个内部`store._vm`的`computed`上，然后定义`store.getters`公共`api`供业务代码使用,这样就很巧妙的利用了`Vue computed`的懒惰计算了，每个`getter`只有在内部的依赖发生变化时才会重新计算，进而业务代码的相关属性也会享受到懒惰计算的好处。

更新`vm store` ， 在`store`创建最初用来初始化`vm store` , 并且保存在`Vue`实例上的

```javascript
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm;
  // bind store public getters
  store.getters = {};
  // reset local getters cache
  store._makeLocalGettersCache = Object.create(null);
  const wrappedGetters = store._wrappedGetters;
  const computed = {};
  forEachValue(wrappedGetters, (fn, key) => {
    // 使用computed来利用其延迟缓存机制 , 直接内联函数的使用将导致关闭保留oldVm
    // 使用局部返回函数，仅保留在封闭环境中保留的参数
    computed[key] = partial(fn, store);
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    });
  });

  // 使用Vue实例存储状态树 , 禁止警告，以防用户添加了一些时髦的全局混合
  const silent = Vue.config.silent;
  Vue.config.silent = true;
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  });
  Vue.config.silent = silent;
  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store);  //严格模式下会调用store._vm.$watch监听
  }
  if (oldVm) {
    if (hot) {
      // 在所有订阅的观察者中调度更改，以强制对getters进行重新评估以进行热加载。
      store._withCommit(() => {
        oldVm._data.$$state = null;
      });
    }
    //Vue下一个生命周期来临时destroy
    Vue.nextTick(() => oldVm.$destroy());
  }
}
```

##### `mutation`的调用

在`mutation`注册完成之后 , 可以进行调用来对`store 的 state `进行更改

```javascript
const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})
store.commit('increment')  //1
```

`store.commit`

```javascript
commit (_type, _payload, _options) {
    // check object-style commit
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    const entry = this._mutations[type]
    if (!entry) {
      return
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    /**
    **  _withCommit (fn) {
    **    const committing = this._committing
    **    this._committing = true
    **    fn()
    **    this._committing = committing
    ** }
    **/        
    this._subscribers
      .slice() // 如果订户同步调用退订，则浅拷贝副本可防止迭代器失效
      .forEach(sub => sub(mutation, this.state))
  }
```

##### `action`的分发机制---`dispatch`

虽然`action`的结果就是提交了`mutation` , 但是对于`mutation`而言 ， 他是必须同步执行的 ， 但是对于`action`来说是异步的

```javascript
//官方实例
actions: {
  checkout ({ commit, state }, products) {
    // 把当前购物车的物品备份起来
    const savedCartItems = [...state.cart.added]
    // 发出结账请求，然后乐观地清空购物车
    commit(types.CHECKOUT_REQUEST)
    // 购物 API 接受一个成功回调和一个失败回调
    shop.buyProducts(
      products,
      // 成功操作
      () => commit(types.CHECKOUT_SUCCESS),
      // 失败操作
      () => commit(types.CHECKOUT_FAILURE, savedCartItems)
    )
  }
}
```

```javascript
dispatch (_type, _payload) {
    // check object-style dispatch
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      return
    }

    try {
      this._actionSubscribers
        .slice() 
        .filter(sub => sub.before)
        .forEach(sub => sub.before(action, this.state))
    } catch (e) {
    }

    const result = entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)

    return new Promise((resolve, reject) => {
      result.then(res => {
        try {
          this._actionSubscribers
            .filter(sub => sub.after)
            .forEach(sub => sub.after(action, this.state))
        } catch (e) {
        }
        resolve(res)
      }, error => {
        try {
          this._actionSubscribers
            .filter(sub => sub.error)
            .forEach(sub => sub.error(action, this.state, error))
        } catch (e) {
        }
        reject(error)
      })
    })
  }
```

用`promise.all`执行所有的`action` , 返回的仍然是`promise`就是为了方便链式调用