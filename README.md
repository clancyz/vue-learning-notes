# Vue learing notes

My note of learning Vue.js

## 0.1.0



#### initial setup [83fac01](https://github.com/vuejs/vue/tree/83fac017f96f34c92c3578796a7ddb443d4e1f17)

`Vue` 最初的名字叫 `element` 。
    
---
    
#### rename [871ed91](https://github.com/vuejs/vue/tree/871ed9126639c9128c18bb2f19e6afd42c0c5ad9)

改名叫 `seed` ，初期思路的尝试。
    
对`Mustache`语法 `{{}}` 做一个replace,替换掉原来的innerHTML

```html
<p>{{msg}}</p>
// parse后：
<p><span data-element-binding="msg"></span></p>
```

同时维护一个 `bindings` 变量，存储所有 `{{}}` 中的绑定变量，如 `msg`

对 `msg` 维护一个`els`的属性，使用 `document.queryselectAll("[data-element-binding=*]")`  拿到所有DOM，存入。然后删除 `data-element-binding` 这个属性。

使用 `Object.defineProperty` 中的 `set` 钩子完成`值 -> dom` 的**单向绑定**。

```js
Object.defineProperty(data, variable, {
    set: function (newVal) {
        // 赋值时设定binding, 改动dom
        [].forEach.call(bindings[variable].els, function (e) {
            bindings[variable].value = e.textContent = newVal
        })
    },
    get: function () {
        return bindings[variable].value
    }
})
```
    
---

- naive implementation [a5e27b1](https://github.com/vuejs/vue/commit/a5e27b1174e9196dcc9dbb0becc487275ea2e84c)
 
初期的实现，作者当时ms在参考 [Rivets.js](https://github.com/mikeric/rivets) 

**filters.js**

`filters` 来对绑定变量做特殊处理，这里仅是一个 `capitalize` 。

**directive.js**

`directives` 一些内置指令，是 `vue` 使用者很熟悉的指令：
- text: el.textContent
- show: el.style.display
- class: add/remove className
- on: update, unbind, customFilter 

**main.js**

`new Seed(opts)`做初始化，参数`opts`是根dom。

在根dom里面把为具有`sd`的`prefix`，且满足在当前 `内置directives` 中条件的dom 找出来。比如 

```html
<div id="test" sd-on-click="changeMessage | .button">
    <p sd-text="msg | capitalize"></p>
    <p sd-show="something">YOYOYO</p>
    <p class="button" sd-text="msg"></p>
    <p sd-class-red="error" sd-text="hello"></p>
</div>
```

整个流程如下
    
    
```
graph TB
A(取出所有内置指令)-->B(获取root);
B-->C(解析并parse该内置指令,设置update);
C-->D(在seed scope中绑定所有key);
D-->E(对所有key做get/set处理)

```


同时，在 `set` 函数执行时, 由于内置指令的预定义函数绑定了directive的update。(见函数`bindAccessors`)，
所以统一调用 `directive.update` 方法。
不同的内置指令对应不同的行为，如：

```js
{
    text: function (el, value) {
        el.textContent = value || ''
    },
    show: function (el, value) {
        el.style.display = value ? '' : 'none'
    }
}
```

最后，从`TODO`中可以了解作者的思路：

- nested levels by parsing dot syntax (which means nested getter/setters...)

例如在传

```js
new Seed(dom, {
    'msg.wow': 'hello'
})
```

的情况下，即有nested getter/setter

- repeat directive by watching an array： 这个应该是 `v-for` 之类的遍历数组了
- parse textNodes：对TextNode进行特殊处理，比如{{hello}},转化成`dom attribute`效率还是比较低
- make Seeds compositable：可以mixin?
- formatter arguments: 不知道要干嘛
- Seed.extend(): 创建一个新实例
- options to pass in templates to Seed.create()：new Seed(arguments)

    
---

#### filter value should not be written [3eb7f6f](https://github.com/vuejs/vue/commit/3eb7f6fc72f40ac43aa502e2e8cf70404a490b11)

#### dump, destroy, fix filters [ec39439](https://github.com/vuejs/vue/commit/ec394395569ea81e50209b9e241e789cfacde588)

修了个bug,就是用户原始传入的值被`filter`处理后，是需要保持原值的，`filter`处理的是值的副本。

`Seed`构造函数加入 `dump` 和 `destroy`

---


#### refrator [cf1732b](https://github.com/vuejs/vue/commit/cf1732bea21dcc1637d587d295d534535a92d2b7)

把内置指令的parse/具体方法划分为 `directive.js`和 `directives.js`

做了 `this` 的绑定

`class`和 `repeat` 暂时还没有实现 

---

#### augmentArray seems to work [154861f](https://github.com/vuejs/vue/commit/cf1732bea21dcc1637d587d295d534535a92d2b7)

覆写了`Array.prototype.push`方法，push时进行mutate,做到**数组长度改变时的监听**

```js
function augmentArray (collection, directive) {
    collection.push = function (element) {
        push.call(this, arguments)
        directive.mutate({
            event: 'push',
            elements: slice.call(arguments),
            collection: collection
        })
    }
}
```

再看这一段

```
<div id="test" sd-on-click="changeMessage | delegate .button">
...
</div>
```

filter.js里面使用了`Element.prototype.matches` 来进行`事件代理` ，但是这里有bug, `e.target !== e.currentTarget` 时需要往`parentNode`上继续检测的嘛，而且`matches`有兼容性问题，待后续看解决方案是`polyfill`还是其他。

```js
delegate: function (handler, selectors) {
        return function (e) {
            var match = selectors.every(function (selector) {
                return e.target.webkitMatchesSelector(selector)
            })
            if (match) handler.apply(this, arguments)
        }
    }
```

---

#### WIP [79760c0](https://github.com/vuejs/vue/commit/79760c09d50caca7ca27cd85991eb2c6e9ba3231)

抽离设置到`config.js`
`filter.js` 当filter函数没定义时throw Error

数组的监听方面，监听了几个常用的方法, 做本职工作的同时加入`callback`

```js
var proto = Array.prototype,
    slice = proto.slice,
    mutatorMethods = [
        'pop',
        'push',
        'reverse',
        'shift',
        'unshift',
        'splice',
        'sort'
    ]

module.exports = function (arr, callback) {

    mutatorMethods.forEach(function (method) {
        arr[method] = function () {
            proto[method].apply(this, arguments)
            callback({
                event: method,
                args: slice.call(arguments),
                array: arr
            })
        }
    })

}
```

`main.js` 只作为一个入口点，主要逻辑移到了`seed.js`


---

#### refractor [5ce3b82](https://github.com/vuejs/vue/commit/5ce3b82b91a134f57dd1dffe8553cec369a56c70)

`sd-repeat` --> `sd-each`

引入`Seed.extend` 替换原来的 `Seed.seed`（ms原来这种命名感觉会绕晕），通过原型继承创建子组件

作者打算写一个`todo` , 毕竟是MVVM的标准Demo...

---

#### kinda working now [952ab43](https://github.com/vuejs/vue/commit/952ab433a5a37f56ce95b06578b028ecdfdcad6c)

作者在`todo.md`里面做的思考：

- ? separate scope data and prototype methods // think about this

`MVVM` 里面不可避免的会碰上 `nested scope` 。 如： 

```html
<parent>
  <child>
    <button sd-on-click="change" sd-text="hello">
  </child>
</parent>
```

那么问题是：比如parent 和 child 都有`change`方法，data中都有`hello`，应该用哪个？遍历dom树解析的时候如何解析？

假设我们希望`hello`是只读取`child`的,比较合情理。 

那么有N个`child` 绑定 `change`事件呢？我们又希望能够做`delegate`绑到`parent`上去.....

那么如何实现这样的功能，甚至是一个 `isolated scope` ？

这个任何一个`MVVM`框架都会做而且必须做。

---

#### sd-each-* works [3149839](https://github.com/vuejs/vue/commit/31498397366dc036911690e06670a1b0d1746654)

实现了数组循环绑定。

```
<li sd-each-todo="todos">
    <span class="todo" sd-text="todo.title" sd-class-done="todo.done"></span>   
</li>
```

要实现循环绑定，那么按上面例子就需要插入n个span. 这里引入了一个`childSeed` ,当长度大于0时进行插入：

```js
function buildItem (data, index, collection) {
    var node = this.el.cloneNode(true),
        spore = new Seed(node, data, {
            eachPrefixRE: this.prefixRE,
            parentScope: this.seed.scope
        })
    this.container.insertBefore(node, this.marker)
    collection[index] = spore.scope
    return spore
}
```

在`Seed.prototype._compileNode`中对`textnode`进行处理, `_compileTextNode`暂还未实现：

```js
if (node.nodeType === 3) {
    // text node
    self._compileTextNode(node)
}
```

#### big refactor.. again [23a2bde](https://github.com/vuejs/vue/commit/23a2bde88917cd3f043beca4f5da3b37b0f9de29)

重构狂魔的作者又搞出来一个`controller`的概念。。是否也受了ng的影响。。

`controller` 主要就是解决 `scope nesting`的问题，这个版本还没实现

实际就是创建一个`Seed`新实例, 避免混淆

```js
Seed.controller = function (id, extensions) {
    if (controllers[id]) {
        console.warn('controller "' + id + '" was already registered and has been overwritten.')
    }
    var c = controllers[id] = Seed.extend(extensions)
    return c
}
```

原先的 `directive.js` 和 `directives.js` 显然看名字有点晕，所以改成了`binding.js`

前面提到的：

> filter.js里面使用了`Element.prototype.matches` 来进行`事件代理` ，但是这里有bug, `e.target !== e.currentTarget` 时需要往`parentNode`上继续检测的嘛，而且`matches`有兼容性问题，待后续看解决方案是`polyfill`还是其他。

`filter.js`中的`deletegate`方法做了一个`delegateCheck` ，做冒泡遍历，修掉之前bug。

`Seed.bootstrap` 对于`controller`做一个自检测，但是还未加入流程。

---

#### kinda work now except for scope nesting [62b75d4](https://github.com/vuejs/vue/commit/62b75d49838744f8b865064256c5f0f8df51616e)

#### almost there! scope nesting [15ffaa4](https://github.com/vuejs/vue/commit/15ffaa4166f65ae62eb3ac64fa0ca83e1455bda2)

`_compileTextNode` 方法做了实现：先match, 如果自己没有，去找`parentSeed`

```
<ul sd-show="todos">
    <li class="todo"
        sd-controller="Todo"
        sd-each="todo:todos"
        sd-class="done:todo.done"
        sd-on="click:changeMessage, click:todo.toggle"
        sd-text="msg"
    ></li>
</ul>
```

里面对于`nested scope`的实现：

```js
var key = bindingInstance.key,
        epr = this._options.eachPrefixRE,
        isEachKey = epr && epr.test(key),
        scopeOwner = this
    // TODO make scope chain work on nested controllers
    if (isEachKey) {
        key = key.replace(epr, '')
    } else if (epr) {
        scopeOwner = this._options.parentSeed
    }
```

感觉现在这样的设计用起来还是有点囧的...必须要标注一下自己是属于哪个scope的

---

#### finally workssssss [08e7992](https://github.com/vuejs/vue/commit/08e7992c2a7fd0f686d2bf22205822dacf74b53f)

还在解决 `nested scope`的问题。

#### milestone reached, update todo [83665f9](https://github.com/vuejs/vue/commit/83665f9962d041378213bffb61647eff25e67a46)

一个基本的`MVVM`已经实现了。但是原来的`nested scope`实现被作者自己否了。

```js
else if (ctrlExp && !root) { // nested controllers
    // TODO need to be clever here!
} 
```

`todo.md`

- nested controllers - but how to inherit scope? 作者在思考作用域继承的问题。
- improve arrayWatcher 
- parse textNodes 
- computed properties : computed properties的萌芽，天才的想法~

---

















 

    








