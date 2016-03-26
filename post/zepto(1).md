``zepto``主要代码框架：

```javascript
var Zepto = (function () {
  var zepto = {},
      fragmentRE = /^\s*<(\w+|!)[^>]*>/

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