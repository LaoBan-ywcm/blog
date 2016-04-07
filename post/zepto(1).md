``zepto``主要代码框架：

```javascript
var Zepto = (function () {
  var zepto = {}

  function Z(dom, selector) {
    var i, len = dom ? dom.length : 0
    for (i = 0; i < len; i++) this[i] = dom[i]
    this.length = len
    this.selector = selector || ''
  }

  zepto.Z = function(dom, selector) {
    return new Z(dom, selector)
  }

  zepto.init = function(selector, context) {
    var dom

    // 省略....

    return zepto.Z(dom, selector)
  }


  $ = function(selector, context){
    return zepto.init(selector, context)
  }

  $.fn = {
    constructor: zepto.Z
  }

  zepto.Z.prototype = Z.prototype = $.fn

  $.zepto = zepto

  return $
})()

window.Zepto = Zepto
window.$ === undefined && (window.$ = Zepto)
```

源码中最重要的有三个：函数``$``, 对象``zepto``, 构造函数``Z``。

可以看到平时用全局``Zepto``或``$``，就是函数自执行中返回的``$``。当调用``$('...')``时，实际是调用了``zepto.init``来根据``selector``的不同来做不同的处理生成``dom``, 再通过``zepto.Z``从而获取一个``Z``的实例。我们将一些属性和平时常用的那些方法写在``$.fn``中，通过``Z.prototype = $.fn``将这些赋给``Z``的实例。而``zepto``更像是``$``和``Z``的中间衔接层。


来看最主要的``zepto.init``：
```
var fragmentRE = /^\s*<(\w+|!)[^>]*>/

zepto.init = function(selector, context) {
  var dom

  if (!selector) return zepto.Z()

  else if (typeof selector == 'string') {
    selector = selector.trim()

    if (selector[0] == '<' && fragmentRE.test(selector))
      dom = zepto.fragment(selector, RegExp.$1, context), selector = null
    else if (context !== undefined) return $(context).find(selector)
    else dom = zepto.qsa(document, selector)
  }

  else if (isFunction(selector)) return $(document).ready(selector)

  else if (zepto.isZ(selector)) return selector

  else {
    // normalize array if an array of nodes is given
    if (isArray(selector)) dom = compact(selector)
    // Wrap DOM nodes.
    else if (isObject(selector))
      dom = [selector], selector = null
    // If it's a html fragment, create nodes from it
    else if (fragmentRE.test(selector))
      dom = zepto.fragment(selector.trim(), RegExp.$1, context), selector = null
    // If there's a context, create a collection on that context first, and select
    // nodes from there
    else if (context !== undefined) return $(context).find(selector)
    // And last but no least, if it's a CSS selector, use it to select nodes.
    else dom = zepto.qsa(document, selector)
  }
  // create a new Zepto collection from the nodes found
  return zepto.Z(dom, selector)
}
```
当``selector``为空时，返回一个``dom``为空的``Z``的实例。

当``selector``为``string``时，有三种情况：
1. 当``selector``为标签时，``RegExp.$1``为元素名或者``!``，调用``zepto.fragment``：
```
var singleTagRE = /^<(\w+)\s*\/?>(?:<\/\1>|)$/,
    tagExpanderRE = /<(?!area|br|col|embed|hr|img|input|link|meta|param)(([\w:]+)[^>]*)\/>/ig,
    table = document.createElement('table'),
    tableRow = document.createElement('tr'),
    containers = {
      'tr': document.createElement('tbody'),
      'tbody': table, 'thead': table, 'tfoot': table,
      'td': tableRow, 'th': tableRow,
      '*': document.createElement('div')
    },
    // special attributes that should be get/set via method calls
    methodAttributes = ['val', 'css', 'html', 'text', 'data', 'width', 'height', 'offset']

function isWindow(obj)     { return obj != null && obj == obj.window }
function isObject(obj)     { return type(obj) == "object" }
function isPlainObject(obj) {
  return isObject(obj) && !isWindow(obj) && Object.getPrototypeOf(obj) == Object.prototype
}

zepto.fragment = function(html, name, properties) {
  var dom, nodes, container

  // A special case optimization for a single tag
  if (singleTagRE.test(html)) dom = $(document.createElement(RegExp.$1))

  if (!dom) {
    if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
    if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
    if (!(name in containers)) name = '*'

    container = containers[name]
    container.innerHTML = '' + html
    dom = $.each(slice.call(container.childNodes), function(){
      container.removeChild(this)
    })
  }

  if (isPlainObject(properties)) {
    nodes = $(dom)
    $.each(properties, function(key, value) {
      if (methodAttributes.indexOf(key) > -1) nodes[key](value)
      else nodes.attr(key, value)
    })
  }

  return dom
}
```

```
function likeArray(obj) { return typeof obj.length == 'number' }

$.each = function(elements, callback){
  var i, key
  if (likeArray(elements)) {
    for (i = 0; i < elements.length; i++)
      if (callback.call(elements[i], i, elements[i]) === false) return elements
  } else {
    for (key in elements)
      if (callback.call(elements[key], key, elements[key]) === false) return elements
  }

  return elements
}
```