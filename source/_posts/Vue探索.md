---
title: Vue探索
date: 2020-09-09 22:39:34
tags: Vue
category: Vue
---

# `Vue`学习

## `Vue`源码学习

### 从 `new Vue()`开始

![](E:\coding\myWebSite\source\_posts\Vue探索\newVue.png)

`vue`其实只是一个`class / function`

```javascript
function Vue (options) {
  this._init(options)
}
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)     //给数据流劫持和绑定
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created'

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

Vue 初始化主要就干了几件事情，合并配置，初始化生命周期，初始化事件中心，初始化渲染，初始化 data、props、computed、watcher 等等。

### `Vue`挂载

```javascript
//首先将Vue原型链上的mount缓存起来 , 在之后在对mount方法进行重写
/**
// public mount method  最原始的Vue的mount方法
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
**/
//缓存的原因是方便在之后对其进行调用
//关于用将Vue.prototype.$mount 赋值给mount，而后修改Vue.prototype.mount  mount并没有发生改变 ，其实应该是对象中的key所存储的也应该是value所对应的指针
//关于js中Object的实现可以看看https://zhuanlan.zhihu.com/p/26169639
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)
    //限制Vue不挂载document.body ， document.documentElement
  if (el === document.body || el === document.documentElement) {
    return this
  }

  const options = this.$options
  // 如果options没有render方法，就将template/el转化为render方法
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
        }
      } else if (template.nodeType) {
        //node.nodeType  返回一个整数，其代表的是节点类型。具体参考https://developer.mozilla.org/zh-CN/docs/Web/API/Node/nodeType#%E8%8A%82%E7%82%B9%E7%B1%BB%E5%9E%8B%E5%B8%B8%E9%87%8F  
        template = template.innerHTML  //获得他所有的子元素
      } else {
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)    //在DOM上获取到元素
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
 /** 这里的compileToFunctions  函数实际上对应的是src/complier/index.js里面的
 export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
 //生成Ast树，
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
//这里对应着Vue的真实的render函数
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
**/
      options.render = render
      options.staticRenderFns = staticRenderFns
    }
  }
  return mount.call(this, el, hydrating)
}
```

实际上`init`函数里面调用的`mount`执行的是最开始缓存起来的`mountComponent`方法

#### `mountComponent`

```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }

  // 
  new Watcher(vm, updateComponent, noop, {  //Watcher监听数据的变化，初始化执行回调函数，绑定数据与DOM之间的关系 ，之后就是数据变化后 ， 会执行updateComponent中的vm._render进行数据更新
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // 
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

### `Vue`渲染

```javascript
 Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    if (_parentVnode) {
      vm.$scopedSlots = normalizeScopedSlots(
        _parentVnode.data.scopedSlots,
        vm.$slots,
        vm.$scopedSlots
      )
    }
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      currentRenderingInstance = vm
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render`)
      if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
      } else {
        vnode = vm._vnode
      }
    } finally {
      currentRenderingInstance = null
    }
    if (Array.isArray(vnode) && vnode.length === 1) {
      vnode = vnode[0]
    }
    if (!(vnode instanceof VNode)) {
      vnode = createEmptyVNode()
    }
    vnode.parent = _parentVnode
    return vnode
  }
```

在 Vue 的官方文档中介绍了 `render` 函数的第一个参数是 `createElement`，那么结合之前的例子：

```html
<div id="app">
  {{ message }}
</div>
```

相当于我们编写如下 `render` 函数：

```javascript
render: function (createElement) {
  return createElement('div', {
     attrs: {
        id: 'app'
      },
  }, this.message)
}
```

`render`的过程就是构建一个虚拟`DOM`的过程

#### 虚拟`DOM`

虚拟`DOM`的class在`src\core\vdom\vnode.js`下面 , 给定`Vue`中的虚拟`DOM`的属性和元素

产生虚拟`DOM`的原因主要还是因为`DOM`元素在浏览器的是非常大而且复杂的，如果频繁的进行更新操作，就会非常影响性能，而简单的用`js`对象来描述一个`DOM`元素，是非常轻量和简单的。包括在`react`中也是一样的

#### `update`将虚拟`DOM`改为真实`DOM`

`Vue.prototype._update = function`在`src\core\instance\lifecycle.js`下

`update`函数判断有无虚拟树 ， `_update` 的核心就是调用 `vm.__patch__` 方法

`update`既有在初始时将虚拟`DOM`变为真实地`DOM`之外也会进行负责数据的更新

```javascript
const prevVnode = vm._vnode

if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
```

`update`中的最主要的方法就是`vm.__patch__` , 而`vm.__patch__`会根据你的环境的不同，使用不同的方法对虚拟`DOM`进行映射到提交到不同的平台上`render`



## 简单实现`Vue`

```javascript
//简单的实现VUE的双向绑定
//es6语法
//vue的双向绑定的基本原理 ， 用的就是Object.defineProperty中定义的get , set属性
//首先v-bind ， v-onclick ， v-model 在元素上的属性就是通过语法解析 ， 然后在对元素进行相应监听事件或者数据的绑定 ， 即使的通知到数据的更新

class myVue{
    constructor(options){
        this._init(options)
    }

    _init(options){  //初始化，执行model与view的绑定
        this.$options = options
        const {el , data , methods} = this.$options
        this.$el = document.querySelector(el)                     //选取页面的父节点
        this.$data = data
        this.$methods = methods

        this.binding = {}   //存储dom元素与vm.data之间的映射关系

        const vm = this

        _obverse(this , data)
        complie(this.$el , vm)
    }
}

//对data的数据劫持完成之后需要一个Watcher,用来绑定更新函数,实现对DOM元素的更新
class Watcher{
    //name     指令名称，例如文本节点则为'text'
    //el       对象的dom元素
    //vm       myVue实例
    //exp      执行对应的值
    //attar    绑定的属性 ， 例如'innerText'
    constructor(name , el , vm , exp , attar){
        this.name = name
        this.el = el
        this.vm = vm
        this.exp = exp
        this.attar = attar
    }
    update(){
        this.el[this.attar] = this.vm.$data[this.exp]
    }
}

//对于data的get ， set属性进行重写
function _obverse(vm , data){
    const binding = vm.binding

    const keys = Object.keys(data)
    for(let key of keys){
        console.log(key , binding)
        if(data.hasOwnProperty(key)){
            binding[key] = {
                _directives : []
            }
            let value = vm.$data[key]
            if(typeof value === 'object'){
                _obverse(vm , value)
            }
            Object.defineProperty(vm.$data, key, {
                enumerable: true,
                configurable: true,
                get: function () {
                  console.log(`${key}获取${value}`);
                  return value;
                },
                set: function (newVal) {
                  console.log(`${key}更新${newVal}`);
                  if (value !== newVal) {
                    value = newVal;
                    binding[key]._directives.forEach(function (item) {
                      item.update();
                    })
                  }
                }
              })
        }
    }
}

//最后需要一个函数进行解析语法
function complie(root , vm){
    //拿到它的子元素
    const nodes = root.children
    console.log(nodes , nodes.length)
    for(let i = 0 ; i < nodes.length ; i++){
        let node = nodes[i]
        console.log(node , node.children.length , node.children , i)
        //对每一个子元素的语法进行相应的语法添加
        if(node.children.length){
            complie(node , vm)
        }
        if(node.hasAttribute('v-click')){
            node.onclick = function(){
                const attrVal = nodes[i].getAttribute('v-click');
                return vm.$methods[attrVal].bind(vm.$data)();
            }
        }
        if(node.hasAttribute('v-model') && (node.tagName = 'INPUT' || node.tagName == 'TEXTAREA')){
            const attar = node.getAttribute('v-model')
            node.addEventListener('input' , (function(key){
                vm.binding[attar]._directives.push(new Watcher(
                    'input',
                    node,
                    vm,
                    attar,
                    'value'
                ))
                return function(){
                    vm.$data[attar] = nodes[key].value
                }
            })(i))
        }
        if(node.hasAttribute('v-bind')){
            const attar = node.getAttribute('v-bind')
            console.log(attar , vm.binding)
            vm.binding[attar]._directives.push(new Watcher(
                'text',
                node,
                vm,
                attar,
                'innerHTML'
            ))
        }  
    }
}
```

