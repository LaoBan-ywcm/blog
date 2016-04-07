## index.js
```javascript
import Vue from './instance/vue'

Vue.version = '1.0.20'

export default Vue
```
从入口文件可以看到，核心在``instance``。


## instance
``instance/vue.js``中最重要的就是这个构造函数:
```javascript
function Vue (options) {
  this._init(options)
}
```
公开的API: 属性和方法以``$``为前缀；私有的属性和方法以``_``为前缀。

将实例``APIS``的定义都放在``api``文件夹，内部的定义都放在``internal``文件夹。其实每一个文件都是给``Vue.prototype``增加属性和方法。

```javascript
import initMixin from './internal/init'
import stateMixin from './internal/state'
import eventsMixin from './internal/events'
import lifecycleMixin from './internal/lifecycle'
import miscMixin from './internal/misc'

import dataAPI from './api/data'
import domAPI from './api/dom'
import eventsAPI from './api/events'
import lifecycleAPI from './api/lifecycle'
```
这些文件都会``export``一个以构造函数``Vue``为参数的函数：
```javascript
// install internals
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
miscMixin(Vue)

// install instance APIs
dataAPI(Vue)
domAPI(Vue)
eventsAPI(Vue)
lifecycleAPI(Vue)

export default Vue
```
我们来一个个的看。

#### instance/internal/init.js
上面``Vue``的构造函数中用了一个``_init``方法来接受``options``实例化，这个方法就是在这个文件中定义的:
```javascript
import { mergeOptions } from '../../util/index'

let uid = 0

export default function (Vue) {
  Vue.prototype._init = function (options) {
    // options 就是我们传给构造函数的参数项
    options = options || {}

    // 下面这些都是实例的属性
    // 挂载元素
    this.$el = null
    // 父实例
    this.$parent = options.parent
    // 当前组件树的根 Vue 实例, 如果当前实例没有父实例，值将是它自身
    this.$root = this.$parent
      ? this.$parent.$root
      : this
    // 当前实例的直接子组件
    this.$children = []
    // 一个对象，包含注册有 v-ref 的子组件。 v-ref：在父组件上注册一个子组件的索引，便于直接访问。不需要表达式。必须提供参数 id。可以通过父组件的 $refs 对象访问子组件。
    this.$refs = {}       // child vm references
    // 一个对象，包含注册有 v-el 的 DOM 元素。v-el 可以为 dom 元素注册一个索引。通过所属实例的 $els 就可以访问这个元素。
    this.$els = {}        // element references
  }
}
```
自己看``vue``源码，参照了赵锦江(勾三股四)的一篇``vue``源码总结文章，其中有一段总结``_init``：
> 整个实例初始化的过程中，重中之重就是把数据 (Model) 和视图 (View) 建立起关联关系。Vue.js 和诸多 MVVM 的思路是类似的，主要做了三件事：
1. 通过 observer 对 data 进行了监听，并且提供订阅某个数据项的变化的能力
2. 把 template 解析成一段 document fragment，然后解析其中的 directive，得到每一个 directive 所依赖的数据项及其更新方法。比如 v-text="message" 被解析之后 (这里仅作示意，实际程序逻辑会更严谨而复杂)：
  * 所依赖的数据项 this.$data.message，以及
  * 相应的视图更新方法 node.textContent = this.$data.message
3. 通过 watcher 把上述两部分结合起来，即把 directive 中的数据依赖订阅在对应数据的 observer 上，这样当数据变化的时候，就会触发 observer，进而触发相关依赖对应的视图更新方法，最后达到模板原本的关联效果。
所以整个 vm 的核心，就是如何实现 observer, directive (parser), watcher 这三样东西

我们接着看``_init``内部：
```javascript
// 实例所有的watcher
this._watchers = []   
// 实例所有的directive（指令）
this._directives = []

// 用于唯一标示vue实例
this._uid = uid++

// 这个标志用来避免实例被 observe
this._isVue = true

// 注册的 callbacks
this._events = {}         
// 这个不清楚，后面看
this._eventsCount = {}       // for $broadcast optimization

// fragment instance properties
this._isFragment = false
this._fragment =         // @type {DocumentFragment}
this._fragmentStart =    // @type {Text|Comment}
this._fragmentEnd = null // @type {Text|Comment}

// 生命周期的状态
this._isCompiled =
this._isDestroyed =
this._isReady =
this._isAttached =
this._isBeingDestroyed =
this._vForRemoving = false
this._unlinkFn = null
```
下面三个等看到后面，再来理解:
```javascript
// context:
// if this is a transcluded component, context
// will be the common parent vm of this instance
// and its host.
this._context = options._context || this.$parent

// scope:
// if this is inside an inline v-for, the scope
// will be the intermediate scope created for this
// repeat fragment. this is used for linking props
// and container directives.
this._scope = options._scope

// fragment:
// if this instance is compiled inside a Fragment, it
// needs to reigster itself as a child of that fragment
// for attach/detach to work properly.
this._frag = options._frag
if (this._frag) {
  this._frag.children.push(this)
}
```
继续：
```javascript
// 将实例自身放入父实例  / transclusion host
if (this.$parent) {
  this.$parent.$children.push(this)
}

// merge options.
// 这里有一点不明白 constructor.options在哪啊，没见到，等后面看吧
options = this.$options = mergeOptions(
  this.constructor.options,
  options,
  this
)

// 设置 ref
this._updateRef()

// 初始化 data 为一个空对象，这个会在_initScope中被填充
this._data = {}

// 在 merge 前保存构造函数的 data，这样我们可以知道哪些属性在初始化时提供
this._runtimeData = options.data

// 调用生命周期钩子:init（在实例开始初始化时同步调用。此时数据观测、事件和 watcher 都尚未初始化）
this._callHook('init')

// 初始化数据观察 and scope inheritance.
this._initState()

// 初始化事件
this._initEvents()

// 调用生命周期钩子:created
// 在实例创建之后同步调用。此时实例已经结束解析选项，这意味着已建立：数据绑定，计算属性，方法，watcher/事件回调。但是还没有开始 DOM 编译，$el 还不存在。
this._callHook('created')

// 如果提供了挂载元素，那么就开始编译
if (options.el) {
  this.$mount(options.el)
}
```
