# Vue源码阅读-从入口开始的运行流程

Vue 版本 vue2.5.17-beta

如果 Vue 项目的 main.js 文件向下面这样，```import Vue from 'vue'``` 后实例化，那 Vue.js 内部是如何执行的了，下面我们就来一探究竟吧！

```js
import Vue from 'vue'
import App from './App.vue'

Vue.config.productionTip = false

new Vue({
  render: h => h(App),
}).$mount('#app')

```
在 main.js 顶部引入 vue 的那一刻就像打开了潘多拉的盒子，Vue.js 内部会做一系列的初始化，首先是```Vue.prototype```原型对象上绑定一堆方法，接着就是 Vue 构造函数本身绑定全局方法和属性（也可以叫静态方法），在然后入口文件的```$mount```方法绑定到```Vue.prototype```原型对象上，最后```new Vue()```时调用构造函数，给vm实例对象初始化一些属性。

为了更好的理解 Vue，我选择从 runtime+compiler 这个版本的入口文件开始。如果对 Vue 的构建和版本还不是很了解就看我上篇[源码目录与构建](https://juejin.cn/post/6909382983283982343)吧！入口文件路径```src/platforms/web/entry-runtime-with-compiler.js```

### new Vue()

main.js 中引入的 Vue 是个构造函数，顺着入口文件 ``` entry-runtime-with-compiler.js ``` 一路追溯我们回来到 Vue 构造函数的定义位置 ``` src/core/insatnce/index.js ``` ，下方代码是 Vue 构造函数本尊

追溯过程是这样的：
* ```entry-runtime-with-compiler.js ``` 中 ```import Vue from './runtime/index'```
* ```runtime/index.js``` 中 ```import Vue from 'core/index'```
* ```core/index.js``` 中 ```import Vue from './instance/index'```
* ```instance/index.js``` 中就是 Vue构造函数定义的地方了

```js
....

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options) 
}

initMixin(Vue) // Vue.prototypet添加 _init方法
stateMixin(Vue) // Vue.prototype添加 $data $props $watch 方法
eventsMixin(Vue) // Vue.prototype添加 $on  $once $off $emit 方法
lifecycleMixin(Vue) // Vue.prototype添加 _update $forceUpdate $destroy 方法
renderMixin(Vue) // Vue.prototype添加 $nextTick _render 方法 

export default Vue

```
可以看到 Vue 构造函数内部调用了```_init```方法，这里的```_init```方法就是在调用```initMixin(Vue) ```的时候定义的。

我们知道 main.js 中```new Vue()```就是在调用构造函数，构造函数内部又直接调用了```_init```方法，并且传入我们自己定义的options对象

下面我们来看下 ```_init ``` 方法的实现

```js
// src/core/instance/init.js

export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    ....
    
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

```_init ``` 方法的实现主要是对当前vm实例做了合并配置和初始化的操作，可以重点关注下```initState(vm)```，```initState```内部的实现主要是给 ```data/props``` 做一些初始化操作，然后通过 ```proxy```把data中的属性全部代理到vm实例上，在通过```Object.defineProperty```响应化数据，给对象属性设置 ```getter/setter```，传说中的响应式原理就是从这开始的（响应式原理这块后面章节细聊）

最后在查看```vm.$options```中是否有```el```属性，如果有就调用```vm.$mount(vm.$options.el)```, 如果没有就需要手动调用```$mount```，这里我们是手动调用了```$mount('#app')```

## 挂载 $mount()

```$mount``` 在多个地方有定义，如 ```src/platform/web/entry-runtime-with-compiler.js```、```src/platform/web/runtime/index.js```、```src/platform/weex/runtime/index.js```。因为 ```$mount``` 这个方法的实现是和平台、构建方式都相关的。下面重点解释 ```compiler```版本的

```js
// src/platform/web/entry-runtime-with-compiler.js

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}
```
代码中首先缓存 ```/runtime/index``` 下 ```$mount```, 如果没有定义 render 方法，则会把 el 或者 template 字符串转换成 render 方法。我们这里直接写的 ```render: h => h(App)```，所以直接调用 ```mount.call```，在Vue 2.0 后所有的渲染最终都需要 render 方法，不管我options对象中是定义了el还是template，最终都会通过```compileToFunctions```编译成 render 方法，最终都会调用```/runtime/index``` 下 ```$mount```，下面看下runtime中的$mount实现


```js
// src/platform/weex/runtime/index.js

Vue.prototype.$mount = function (
  el?: string | Element,    // 挂载的元素
  hydrating?: boolean       // 服务端渲染相关参数
): Component {
  el = el && inBrowser ? query(el) : undefined        // query就是document.querySelector方法
  return mountComponent(this, el, hydrating)          // 位于core/instance/lifecycle.js
}
```

runtime中的```$mount```方法其实是调用的```mountComponent```，接着看```mountComponent```方法的实现

```js
// src/core/instance/lifecycle.js

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

  // 渲染watcher，当数据更改，updateComponent作为Watcher对象的getter函数，用来依赖收集，并渲染视图
  let updateComponent
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }

  // 渲染watcher, Watcher 在这里起到两个作用，一个是初始化的时候会执行回调函数
  // ，另一个是当 vm 实例中的监测的数据发生变化的时候执行回调函数
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)

  // 这里注意 vm.$vnode 表示 Vue 实例的父虚拟 Node，所以它为 Null 则表示当前是根 Vue 的实例
  if (vm.$vnode == null) {
    vm._isMounted = true               // 表示这个实例已经挂载
    callHook(vm, 'mounted')      
  }
  return vm
}
```

```mountComponent``` 的主要工作，定义了一个updateComponent函数，updateComponent函数内部在调用vm实例原型上的```_render()```和```_update()```, ```_render()```会返回一个vnode对象，在传给```_update()```创建真实的DOM。
接着创建一个渲染Watcher```new Watcher()```,updateComponent函数传入Watcher对象作为getter函数调用。

这里的Watcher有两个作用，其一是初始化后调用updateComponent，其二就是数据更新后重新执行updateComponent。

## 创建vnode

接上节，调用updateComponent函数，首先执行```vm._render()```返回一个vnode实例，下面进入_render具体实现看看：

```js
// src/core/instanse/render.js

 Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    // reset _rendered flag on slots for duplicate slot check
    if (process.env.NODE_ENV !== 'production') {
      for (const key in vm.$slots) {
        // $flow-disable-line
        vm.$slots[key]._rendered = false
      }
    }

    if (_parentVnode) {
      vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
    }

    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render`)
      // return error render result,
      // or previous vnode to prevent render error causing blank component
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        if (vm.$options.renderError) {
          try {
            vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
          } catch (e) {
            handleError(e, vm, `renderError`)
            vnode = vm._vnode
          }
        } else {
          vnode = vm._vnode
        }
      } else {
        vnode = vm._vnode
      }
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
        warn(
          'Multiple root nodes returned from render function. Render function ' +
          'should return a single root node.',
          vm
        )
      }
      vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
}
```

获取配置对象```vm.$options```的 render 函数，抛开代码中的错误信息判断，核心就是 call 的方式调用了我们传入的 render 函数在传入 ```$createElement```，$createElement 是初始化 vm 实例时绑定到 vm 对象上的，现在移步 createElement 的实现：

```js
export function _createElement (
  context: Component,  // vm实例
  tag?: string | Class<Component> | Function | Object,   //这里的tag是APP组件的options对象
  data?: VNodeData,  // 空
  children?: any, // 空
  normalizationType?: number
): VNode | Array<VNode> {
	....
    
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)  
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children) //拍平成一个一维数组
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```
可以先关注下 children 的处理，对 children 做了些规范化和扁平化的处理，如果 tag 是个字符串像这样 ```div```，直接实例一个 vnode 对象，传入 tag data children 。因为我们这里的 tag 是个组件来着```render: h => h(App)```，并不是字符串，所以走 ```createComponent```的逻辑。 

我们这里的 tag 应该是长下面这样：

* 因为 vue-loader 会帮我们把 .vue 文件的 template 部分编译成 render 方法
```js
{
  name: 'App',
  render() {....},
  components: {
    HelloWorld
  }
}
```

下面转到 createComponent 的实现：

```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  data = data || {}

  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}
```
先明确几点：
* Ctor 是传入的 options 对象
* context 是 vm 实例
* ```context.$options._base``` 是 Vue 构造函数， baseCtor = Vue

如果 Ctor 是个对象就调用 Vue 上的静态方法 ```extend```，创建一个子构造 sub 后续会实例化这个子构造器，接着就是给 data 添加钩子函数，后续会调用钩子函数实例子组件，然后就实例化一个组件 vnode ，data Ctor子构造器 传入构造函数挂在组件 vnode 实例上，方便后续调用， 返回这个组件 vnode ，接着就是调用 ```vm._update``` 传入这 vnode。

## 创建DOM
_update的实现：

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```
update核心是调用 ```__patch__```, 创建真实的DOM，patch的过程就是不停的递归创建DOM，在 appendChild 到父元素上，整个组件patch 的细节我放在后面的章节在讲


patch 的实现：

```js
// core/vdom/patch.js
function patch (oldVnode, vnode, hydrating, removeOnly) {
    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
        // replacing existing element
        const oldElm = oldVnode.elm  // 
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
```

这里的 patch 过程有涉及到了子组件实例化，后续的一系列的子组件实例的初始化，有走到```vm._update(vm._render(), hydrating)``` 创建渲染 vnode， 反正最后会生成一个完整的 DOM 结构后插入 body ，替换掉原先的 ```<div id="app">```，这里的```parentElm```就是 body 


## 总结

整个过程是这样的：


1. ```import Vue from 'vue'```一系列初始化，原型上绑定方法，Vue 函数上绑定方法和属性
2. ```new Vue()``` 调用 ```_init``` 初始化vm实例属性
3. ```$mount```  compiler -> render -> 创建vnode -> _update -> 创建DOM

> 参考
> 1. [Vue.js 技术揭秘](https://ustbhuangyi.github.io/vue-analysis) 
> 2. [Vue.js 文档](https://cn.vuejs.org/)










