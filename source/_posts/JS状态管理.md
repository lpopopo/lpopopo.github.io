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
    this._committing = false
    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()
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

在`store`中的属性`this._modules`原型

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
    }, this.root)
  }

  getNamespace (path) {
  }

  update (rawRootModule) {
  }

  register (path, rawModule, runtime = true) {
    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1) //删除最后一个元素.slice原始数组不会改变)
      parent.addChild(path[path.length - 1], newModule)
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

```javascript
//const state = this._modules.root.state
//installModule(this, state //options, [], this._modules.root)

function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length   //true
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

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = module.context = makeLocalContext(store, namespace, path)

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

