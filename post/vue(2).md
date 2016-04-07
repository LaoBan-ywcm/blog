## instance/internal/state.js
之前在``_init``中，当调用生命周期钩子``init``后，开始数据观察初始化：``this._initState()``。这个方法就是我们现在要看的：
```javascript
export default function (Vue) {
  // $data：Vue 实例观察的数据对象。可以用一个新的对象替换。实例代理了它的数据对象的属性。
  // 当我们设置数据时，都会触发this._setData，这个值得期待，接着看
  Object.defineProperty(Vue.prototype, '$data', {
    get () {
      return this._data
    },
    set (newData) {
      if (newData !== this._data) {
        this._setData(newData)
      }
    }
  })

  // 这个就是文件的核心方法，做了很多初始化
  Vue.prototype._initState = function () {
    this._initProps()
    this._initMeta()
    this._initMethods()
    this._initData()
    this._initComputed()
  }

  // 第一个初始化方法，初始化props
  // props 是期望使用的父组件数据的属性。可以是数组或对象。对象用于高级配置，如类型检查，自定义验证，默认值等。
  Vue.prototype._initProps = function () {
    var options = this.$options
    var el = options.el
    var props = options.props
    // 如果挂载元素不存在并且有props，就会报错
    if (props && !el) {
      process.env.NODE_ENV !== 'production' && warn(
        'Props will not be compiled if no `el` option is ' +
        'provided at instantiation.',
        this
      )
    }
    // 将字符串选择符转化成dom元素，query实际就是用了：document.querySelector
    el = options.el = query(el)

    // 这个后面再看
    this._propsUnlinkFn = el && el.nodeType === 1 && props
      // props must be linked in proper scope if inside v-for
      ? compileAndLinkProps(this, el, props, this._scope)
      : null
  }
}
```
我们再先来看上面的第四个初始化方法，也是非常重要的方法``_initData``，初始化数据：
```javascript
Vue.prototype._initData = function () {
  // data 在组件定义中只能是函数，我们通过它获取数据对象
  var dataFn = this.$options.data
  var data = this._data = dataFn ? dataFn() : {}

  // 数据对象必须是普通对象：原生对象，不推荐观察复杂对象，我们可以看下 isPlainObject
  /**
    var toString = Object.prototype.toString
    var OBJECT_STRING = '[object Object]'
    export function isPlainObject (obj) {
      return toString.call(obj) === OBJECT_STRING
    }
  */
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object.',
      this
    )
  }
  var props = this._props
  // 这里有一个疑问，runtimeData不等于data吗，这个可能只能看第二和三个初始化方法里有没有对data修改
  var runtimeData = this._runtimeData
    ? typeof this._runtimeData === 'function'
      ? this._runtimeData()
      : this._runtimeData
    : null
  // Vue 实例代理数据对象所有的属性
  var keys = Object.keys(data)
  var i, key
  i = keys.length
  while (i--) {
    key = keys[i]
    // 这里有两种情景，我们可以代理一个data的属性
    // 1. 这个属性没有被作为一个prop定义
    // 2. 这个属性通过初始化option提供，并且没有在模版prop中提供

    // 看下hasOwn
    /*
      var hasOwnProperty = Object.prototype.hasOwnProperty
      export function hasOwn (obj, key) {
        return hasOwnProperty.call(obj, key)
      }
    */
    if (
      !props || !hasOwn(props, key) ||
      (runtimeData && hasOwn(runtimeData, key) && props[key].raw === null)
    ) {
      // 代理属性，这个方法下面马上会看到
      this._proxy(key)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        'Data field "' + key + '" is already defined ' +
        'as a prop. Use prop default value instead.',
        this
      )
    }
  }
  // 监听观察 data
  observe(data, this)
}

// 代理一个属性，这样就可以：vm.prop === vm._data.prop
Vue.prototype._proxy = function (key) {
  /*
  * 这个 isReserved 很有意思，它会检查一个字符串是不是以$或_开头
    export function isReserved (str) {
      var c = (str + '').charCodeAt(0)
      return c === 0x24 || c === 0x5F
    }
  */
  // 如果 key 不是以 _ 或 $开头
  if (!isReserved(key)) {
    // need to store ref to self here
    // because these getter/setters might
    // be called by child scopes via
    // prototype inheritance.
    var self = this
    Object.defineProperty(self, key, {
      configurable: true,
      enumerable: true,
      get: function proxyGetter () {
        return self._data[key]
      },
      set: function proxySetter (val) {
        self._data[key] = val
      }
    })
  }
}
```
还记得最开始在``$data``的``setter``中有个``_setData``吗？现在终于到它了：
```javascript
Vue.prototype._setData = function (newData) {
  newData = newData || {}
  var oldData = this._data

  // 这个方法原来会用新的data完全替换旧的
  this._data = newData

  // 现在需要取消代理那些没有出现在新的data中的keys
  var keys, key, i
  keys = Object.keys(oldData)
  i = keys.length
  while (i--) {
    key = keys[i]
    if (!(key in newData)) {
      // 取消代理它，这个方法下面马上就会看到
      this._unproxy(key)
    }
  }

  // 现在还需要代理那些之前没有代理的新的keys，并且触发values的change
  keys = Object.keys(newData)
  i = keys.length
  while (i--) {
    key = keys[i]
    if (!hasOwn(this, key)) {
      this._proxy(key)
    }
  }

  // 卧槽，这个是什么鬼，后面慢慢看吧
  oldData.__ob__.removeVm(this)
  observe(newData, this)
  this._digest()
}

// 取消代理，无需多解释了
Vue.prototype._unproxy = function (key) {
  if (!isReserved(key)) {
    delete this[key]
  }
}

/**
 * Force update on every watcher in scope.
 */
// 这个留着后面来真正理解吧
Vue.prototype._digest = function () {
  for (var i = 0, l = this._watchers.length; i < l; i++) {
    this._watchers[i].update(true) // shallow updates
  }
}
```
其实前面一个有一个很重要的方法``observe``没有提到，下面来看看它吧，但是``_initState``还没有结束，后面会再看。
