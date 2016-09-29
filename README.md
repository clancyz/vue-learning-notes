# Vue learing notes

My note of learning Vue.js

## 0.1.0 （branch 0.10）

#### 初始设置 

把依赖啥的都安装好，`component` 这个全局库**不能**使用最新版本否则执行`grunt task`时会报错 

```
$ npm i -g grunt 
$ npm i -g component@0.16.3
$ npm i
$ component install component/emitter
$ grunt (--force)

```

---


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

filter.js里面使用了`Element.prototype.matches` 来进行`事件代理` ，但是这里有bug, `e.target !== e.currentTarget` 时需要往`parentNode`上继续检测的嘛，~~而且`matches`有兼容性问题，待后续看解决方案是`polyfill`还是其他。~~

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

PS: `Element.matches()` 支持情况, 原则上是可以`加前缀通用`的， 基本都已经支持。即使作`polyfill`也没啥。



Feature | Chrome | Firefox (Gecko) | Internet Explorer | Opera | Safari (WebKit) 
---|---|---|---|---|---
Original support with a non-standard name | Yes | 3.6 (1.9.2) | 9.0 | 11.5 / 15.0 | 5.0 |
Specified version | 34 | 34 | ? | 21.0 | 7.1 |

此数据来自 [mozilla developer](https://developer.mozilla.org/en-US/docs/Web/API/Element/matches)


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

#### emitter [1d7a94e](https://github.com/vuejs/vue/commit/1d7a94ed1abfa5273d6071e1f1f1e2825d32b9a3)

用到了[component/emitter](https://github.com/component/emitter), 让它作为`Seed.prototype` 的 `mixin`,
用来做事件传递


`seed.js`中：
```js
Emitter(Seed.prototype)
```

```js
Seed.controller('Todo', function (scope, seed) {
    scope.toggle = function () {
        scope.done = !scope.done
        seed.parentSeed.emit('toggle', seed)
    }
})
```


---

#### todo demo kinda works [a6b7257](https://github.com/vuejs/vue/commit/a6b72570e22f5cfaaf7bc70e4e7a7dbf18ec23c0)

`Binding` 统一改回 `Directive`

`directives.js` 内置指令中加入了 `sd-checked`

这个demo还不完整，`directives.js`中的`mutate` 中还未加相关逻辑，当数组长度改变时（如添加一个todo）并不能反馈到dom上。

```js

update: function (collection) {
    if (this.childSeeds.length) {
        this.childSeeds.forEach(function (child) {
            child.destroy()
        })
        this.childSeeds = []
    }
    watchArray(collection, this.mutate.bind(this))
    var self = this
    collection.forEach(function (item, i) {
        self.childSeeds.push(self.buildItem(item, i, collection))
    })
},
mutate: function (mutation) {
    console.log(mutation)
}
```

---

#### awesome [fcd9544](https://github.com/vuejs/vue/commit/fcd95440901fd33b7e795e383ea8669a763e3297)

加入了内置指令 `sd-class`


---

#### nested controllers [f1ed54b](https://github.com/vuejs/vue/commit/f1ed54bc84140b083964af12d3fc72c32fc8a085)

作者现在思考在框架中应该包含的内容：

- Template
- Controller
    - Nested Controllers and accessing parent scope
    - Controller inheritance 
- Data
- Data Binding
- Filters
- Computed Properties
- Custom Filter
- Custom Directive

接下来要做的内容：

- complete arrayWatcher

- computed properties (through invoking functions, need to rework the setter triggering mechanism using emitter)
    - （computed properties 基于一个已声明的函数，这里呢就需要在`setter`里面计算依赖啊啥的）

- the data object passed in should become an absolute source of truth, so multiple controllers can bind to the same data (i.e. second seed using it should insert dependency instead of overwriting it)
    - （原始的`data`保留副本，可以让多个实例来用）

- nested properties in scope (kinda hard but try)
    - （nested properties 的确是比较tough的东西）

- parse textNodes



作者做了一下 `nested controllers` 的尝试

```
<div sd-controller="Grandpa">
    <p sd-text="name"></p>

    <div sd-controller="Dad">
        <p><span sd-text="name"></span>, son of <span sd-text="^name"></span></p>

        <div sd-controller="Son">
            <p><span sd-text="name"></span>, son of <span sd-text="^name"></span></p>

            <div sd-controller="Baby">
                <p><span sd-text="name"></span>, son of <span sd-text="^name"></span>, grandson of <span sd-text="^^name"></span> and great-grandson of <span sd-text="$name"></span></p>

            </div>
        </div>
    </div>
</div>
```

作者定义的逻辑是这样的：

> 
> - name -> 自己的scope 
> - ^name -> 父亲的scope
> - ^^name -> 父亲的父亲的scope
> - $name -> root scope 

在这里，先定义了两个正则：

```js
var ancestorKeyRE = /\^/g,
    rootKeyRE = /^\$/
```

这里就是定义个while循环去match例如 `^^name` 这样的东西，找到最终的 `scopeOwner`

```js
if (snr && !isEachKey) {
    scopeOwner = this.parentSeed
} else {
    var ancestors = key.match(ancestorKeyRE),
        root      = key.match(rootKeyRE)
    if (ancestors) {
        key = key.replace(ancestorKeyRE, '')
        var levels = ancestors.length
        while (scopeOwner.parentSeed && levels--) {
            scopeOwner = scopeOwner.parentSeed
        }
    } else if (root) {
        key = key.replace(rootKeyRE, '')
        while (scopeOwner.parentSeed) {
            scopeOwner = scopeOwner.parentSeed
        }
    }
}
```

---
#### solved the setter init invoke [14d0cef](https://github.com/vuejs/vue/commit/14d0cefe6b65e03786eb9811fb39c741d8cd802e)

干掉了统一初始化`scope`的`_dataCopy`属性相关代码, 

```
var binding = {
    value: this.scope[key], // 原先的value是null
    instances: []
}             
```

```
// set initial value
  if (binding.value) {
      directive.update(binding.value) // 初始化时使用这个value
  }
```


---

#### better unbind/destroy [6d81bff](https://github.com/vuejs/vue/commit/6d81bff63a38cd663afffb147d11ca00e38a42be)

调用`unbind(true)`为`destroy`, 调用`unbind()`即为`unbind`

```js
unbind: function (rm) {
    if (this.childSeeds.length) {
        var fn = rm ? 'destroy' : 'unbind'
        this.childSeeds.forEach(function (child) {
            child[fn]()
        })
    }
}
```

改进了`unbind`, 现在是完备的逻辑了

```js
Seed.prototype.unbind = function () {
    var unbind = function (instance) {
        if (instance.unbind) {
            instance.unbind()
        }
    }
    for (var key in this._bindings) {
        this._bindings[key].instances.forEach(unbind)
    }
    this.childSeeds.forEach(function (child) {
        child.unbind()
    })
}
```

作者同时在思考`delegate`的问题。

```js
// TODO probably there's a better way.
// Angular doesn't support delegation either
// any real performance gain?
delegate: function (handler, args) {
    var selector = args[0]
    return function (e) {
        var oe = e.originalEvent,
            target = delegateCheck(oe.target, oe.currentTarget, selector)
        if (target) {
            e.el = target
            e.seed = target.seed
            handler.call(this, e)
        }
    }
}
```

---

#### fix sd-on [8f79a10](https://github.com/vuejs/vue/commit/8f79a10b3684e63b52892caecd3a87d60427aa70)

怒了，把`delegate`整个注掉了。


---

#### computed property progress [3d33458](https://github.com/vuejs/vue/commit/3d33458b601d3c1cf56f0db1e794acc2b161cd2e)

要实现`computed property`就一定需要有`依赖解析`

显然，`依赖解析`最简单的方法是`声明依赖` 

like this:

```
Total: <span sd-text="total < todos"></span> |
Remaining: <span sd-text="remaining"></span> |
Completed: <span sd-text="completed < remaining total"></span>

// computed properties
scope.total = function () {
    return scope.todos.length
}

scope.completed = function () {
    return scope.todos.length - scope.remaining
}
```

上述这种声明方式，声明了`total`依赖`todos`, `completed` 依赖 `remaining` 和 `total`

但是这样对用户来说成本略高。

这个版本中也还未实现调用`total`, `completed`这些声明函数去计算的逻辑。

---

#### event delegation in sd-each [5227248](https://github.com/vuejs/vue/commit/52272487400c22d1829b7a6efbb71483e84a3b7b)

搞回`delegate`, 实现一个*matchSelector 的 `polyfill`

`directives.js` 里面的 `on`,即指令`sd-on` 有以下逻辑

> - bind时，如果是`sd-each`说明是循环，那么需要用`delegate`；定义当前的`selector`及`delegator(父元素)`
> - update时，通过`delegateCheck`找到`delegator`中的`handler`并绑定`eventListener`；



```js
bind: function (handler) {
    if (this.seed.each) {
        this.selector = '[' + this.directiveName + '*="' + this.expression + '"]'
        this.delegator = this.seed.el.parentNode
    }
},
```

但是现在的实现ms是有问题的，因为上面对`this.delegator`赋值的时候，dom还没有渲染完成，这时是没有`parentNode`的

换句话说，`this.delegator`始终为`null`, 绑定的时机不太对。

---


#### break directives into individual files [ca62c95](https://github.com/vuejs/vue/commit/ca62c95a4bbcbc37951eea598c10a88e8e50a549)

拆分了`directives`到不同的文件 

---

#### arrayWatcher [88513c0](https://github.com/vuejs/vue/commit/88513c077d1ca494b075636aa6a6c76257e63f39)


如前所述，对于数组的`mutation`需要做处理。这里定义了一个 `mutationHandlers`，做以下处理

> - push: `self.buildItem` --> `self.container.insertBefore`
> - pop: `pop` 拿出来的元素，当然是直接`$destroy`
> - unshift: `self.buildItem` --> `self.container.insertBefore` --> `self.reorder` 很形象
> - shift: `$destroy` --> `self.reorder` 
> - splice: `$destroy` --> `self.container.insertBefore`
> - sort: `selft.container.insertBefore`

以`push`为例来梳理一下`arrayWatcher`的实现过程：

> - 在scope中执行了push, 进入被watchArray覆写的push方法，执行push
> - 覆写的push方法带着执行mutationHandler， 接着执行mutationHandler[push],传入三个参数：method,args,result
> - 执行 `self.buildItem` , 通过 `cloneNode` 创建一个node
> - 执行 `new Seed()` 创建一个新实例 `spore`, 绑定这个node
> - 把这个`spore` 放到 负责`sd-each` 的 `directive`中的`collection`属性数组中, 这里当然不能再使用push了
> - 执行dom操作 `self.container.insertBefore`

对于 `push` 等不同的函数 `apply` 处理之后，得到的 `result` 显然是不一样的；在`mutationHandlers`里面也对 `result` 进行量身处理。

```js
Object.keys(mutationHandlers).forEach(function (method) {
    arr[method] = function () {
        var result = Array.prototype[method].apply(this, arguments)
        callback({
            method: method,
            args: Array.prototype.slice.call(arguments),
            result: result
        })
    }
})
```

总之这些`arrayWatcher`实现还是挺优雅的。

应该存在的问题是：

- `Array`不止这些方法，是否全搞一次？
- 性能问题

PS:作者的起名还是挺有意思的， `Seed` , `plant`， `spore` ...

---

#### allow overwritting mutationHandlers [343ea29](https://github.com/vuejs/vue/commit/343ea299d0f10d39680b8da6651c3d3ab11ba21c)

`mutationHandlers`可以被覆写。

这是打算弄成api? 按目前的写法显然用户层是不能定义`mutationHandlers`的。而且显然不应该由用户来定义才对啊。。

---

#### better dep parsing [c4f31a5](https://github.com/vuejs/vue/commit/c4f31a5d65ad66fb2bf74612776444648609c0f6)

检测一个`directive`是否有依赖其它`scope data`; 

在这里引入了一个`parseKey` 函数来解析

指令key的写法有几种，比如：

- done:todo.done (sd-class)
- change:toggleTodo (checkbox)
- completed < remaining total （数据的关联计算）

显然现在的声明式依赖只是用`<`符号来声明；于是便有：

```js
var DEPS_RE = /<[^<\|]+/g
```

这里需要避开连续两个`<`符号和声明`filter`的`|`号


如 `completed < remaining total` 这个指令，最终这一段的执行重复利用了`parseKey`, 很优雅：

```js
var depExp = expression.match(DEPS_RE)
    this.deps = depExp
        ? depExp[0].slice(1).trim().split(/\s+/).map(parseKey)
        : null
// depExp === ["< remaining total"]
// depExp[0].slice(1).trim().split(/\s+/)  --> ["remaining", "total"]
// map(parseKey)之后得到this.deps
```


被解析后的结果：

```
this.deps = [{
    arg: null,
    key: 'remaining',
    nesting: false,
    root: false
    }, {
        arg: null,
        key: 'total',
        nesting: false,
        root: false
    }
]
```

接下来是肯定要 do something的：

```
// computed properties
 if (directive.deps) {
        directive.deps.forEach(function (dep) {
            console.log(dep)
        })
    }
```

PS：由用户来声明依赖的方式显然是需要重构的。慢慢看作者的思路。

---

#### computed properties!!! [5acc8a2](https://github.com/vuejs/vue/commit/5acc8a2986c68f9d41baee7878c778d1b47c9f16)

`parseKey`函数中加入了`nesting`和`root`的判断

当依赖（即`directive.deps`）发生变化时调用 `Directive.prototype.refresh` 

```js
// called when a dependency has changed
Directive.prototype.refresh = function () {
    if (this.value) {
        this._update(this.value.call(this.seed.scope))
    }
    if (this.binding.refreshDependents) {
        this.binding.refreshDependents()
    }
}
```

```js
// computed properties
if (directive.deps) {
    directive.deps.forEach(function (dep) {
        var depScope = determinScope(dep, scope),
            depBinding =
                depScope._bindings[dep.key] ||
                depScope._createBinding(dep.key)
        if (!depBinding.dependents) {
            depBinding.dependents = []
            // 每个依赖发生变化时均会触发refresh()
            depBinding.refreshDependents = function () {
                depBinding.dependents.forEach(function (dept) {
                    dept.refresh()
                })
            }
        }
        depBinding.dependents.push(directive)
    })
}
```

对于`watchArray`, 对数组加入了两个辅助方法`replace` 和 `remove`

---

#### todos [dc04a1a](https://github.com/vuejs/vue/commit/dc04a1af6908ce4ad4f89c232c8da0c38a1d92af)

- parse textNodes
- more directives / filters
- nested properties in scope (kinda hard, maybe later)
 
---

#### complete todo example [7d12612](https://github.com/vuejs/vue/commit/7d126127e6c3c234e887c13d35e95aeafdc35de1)

开始战斗力爆表...

照例还是先看`todo`

- parse textNodes?
- method invoke with arguments
- more directives / filters
    - sd-if (终于出来了)
    - sd-route (这是啥)
- nested properties in scope (kinda hard, maybe later)


`todos demo` 大改动，js,css啥的抽出来, 不得不说，css写得还是很有逼格的。。

布局和[todomvc](https://github.com/tastejs/todomvc)差不多。


默认指令添加了 `hide`, `focus`, `value`

`filters`添加了常用的`keyCode`, 加入了`currency` 和 `key` 两个`filter`

---

#### todo [f19e6c3](https://github.com/vuejs/vue/commit/f19e6c36f61ab92e62ded53954ecd93436784b98)

- limited set of expressions (e.g. ternary operator)

考虑在指令中支持更多的`expression`如`三目运算符`

#### thoughts [c1c0b3a](https://github.com/vuejs/vue/commit/c1c0b3a036a110cffd351f8a6394b4e00796f8bc)

`todo.md`中的思考:

- getter setter should emit events
- auto dependency extraction for computed properties (evaluate, record triggered getters)
- use descriptor for computed properties


`getter`和`setter`中做`emit`有啥用？暂时没有想到。。

果然在考虑`自动收集依赖`。。

另外`descriptor`（描述符）是个什么鬼。。

今天先这样吧

---

#### use emitters [9a4e5d0](https://github.com/vuejs/vue/commit/9a4e5d035027d8ad570aca6006dd553c715bcde4)

> `getter`和`setter`中做`emit`有啥用？暂时没有想到。。

恩，在这个commit里面得到了解释，就是使用`Emmiter`来做事件监听啦。以这种`事件驱动`的方式来实现。

例如：

```js
    var seed = this
    Object.defineProperty(this.scope, key, {
        get: function () {
            seed.emit('get', key)
            return binding.value
        },
        set: function (value) {
            if (value === binding.value) return
            seed.emit('set', key, value)
        }
    })

    ...

 // update bindings when a property is set
    this.on('set', this._updateBinding.bind(this))
```

---

#### sourceURLs for dev, reverse value [f6d6bba](https://github.com/vuejs/vue/commit/f6d6bba70d8715a49ced83b8c4db06fec4d2cf79)

grunt的工作，uglify & source mapping 

增加对 `!` 取非操作的逻辑。
 
---

#### add simple example & manual refresh of computed properties [67ff344](https://github.com/vuejs/vue/commit/67ff3448109104fe0d5f6910ba5397fe0a5c6a99)

惯例先来看`todo.md` : 

> - sd-width
> - sd-visible
> - sd-style

弄了一个最简的`hello world example`

加了一个`sd-data`进来，有什么用？只能后面再看了
```js
dataSlt = '[' + config.prefix + '-data]'
```

---

#### auto parse dependency for computed properties!!!!! [7a0172d](https://github.com/vuejs/vue/commit/7a0172d60b636c83a9559b6163d3b2d9058759ab)

作者用了这么多感叹号，想必是很爽的。。实现了`依赖的自动计算`

这个commit对`computed properties`做了较大的重构，需要梳理一下：

以下面这个`total`属性为例：

```js
scope.total = {get: function () {
    return scope.todos.length
}}
```

- `total`依赖于`todos`。
- 把依赖计算写到了`get`函数里。（这里规定了这样的格式）

执行了上面的赋值后，必然触发`set`;

```js
 // add event listener to update corresponding binding
// when a property is set
this.on('set', this._updateBinding.bind(this))
```

由这句，去触发`_updateBinding`, 有以下逻辑：

```js
if (type === 'Object') {
    if (value.get) { // computed property
        this._computed.push(binding)
        binding.isComputed = true
        value = value.get
    } else { // normal object
        // TODO watchObject
    }
} 
```

可以看出：

- 如果`value.get`有定义，那么它就是一个`computed property` （isComputed = true）
- `push`到`Seed`的`_computed`属性中

然后开始：

```js
    // update all instances
    binding.instances.forEach(function (instance) {
        instance.update(value)
    })

    // 接着执行
    if (typeof value === 'function' && !this.fn) {
        // 这里就是开始执行scope.todo.length 了
        value = value()
    }
```

那么在这里又会对`Scope.todo`做`get`操作......


```js

Seed.prototype._createBinding = function (key) {
    ...
    Object.defineProperty(this.scope, key, {
        get: function () {
            if (parsingDeps) {
                // 每当get时，就会触发`get`事件
                depsObserver.emit('get', binding)
            }
            seed.emit('get', key)
            return binding.isComputed
                ? binding.value()
                : binding.value
        },
        ...
    })

    return binding
}

// 接着执行 

    this._computed.forEach(this._parseDeps.bind(this))

// 接着执行 

Seed.prototype._parseDeps = function (binding) {
    // 如上, 触发了`get`事件后，这里的dependents就会收集依赖。
    depsObserver.on('get', function (dep) {
        if (!dep.dependents) {
            dep.dependents = []
        }
        dep.dependents.push.apply(dep.dependents, binding.instances)
    })
    binding.value()
    // 收集完依赖，干掉 
    depsObserver.off('get')
}

```

**这个流程就结束了**





> 如果设置了新值，流程如下：

> update --> refresh --> emitChange -->  所有依赖refresh 

```js
// called when a new value is set
Directive.prototype.update = function (value) {
    ...
    if (this.binding.isComputed) {
        this.refresh()
    }
}

// called when a dependency has changed
Directive.prototype.refresh = function () {
    ...
    this.binding.emitChange()
}

 Binding.prototype.emitChange = function () {
     this.dependents.forEach(function (dept) {
         dept.refresh()
     })
 }
```
---

#### fix _dump() [dada181](https://github.com/vuejs/vue/commit/dada181c8581e4af4e1bf16b86744ebdc8d8d35e)

#### no longer need $refresh [faf0557](https://github.com/vuejs/vue/commit/faf055791e8a0f1b36cdeafb41b13f22eb8b9b8b)

#### clean up, trying to fix delegation after array reset [c0a65dd](https://github.com/vuejs/vue/commit/c0a65ddda5f0e6a332b8dbb2bb2552fdef6f5f89)

结合前面的`get`做的改动。
还在和`delegation`做斗争, 使用`each`指令时, 来操作`parentNode` :

> 那么前面一直存在的各种matchSelector就木用了

```js
bind: function () {
    ...
    this.delegator = this.el.parentNode;
}

 unbind: function (rm) {
     ...
        var delegator = this.delegator
        if (!delegator) return
        var handlers = delegator.sdDelegationHandlers
        for (var key in handlers) {
            console.log('remove: ' + key)
            delegator.removeEventListener(handlers[key].event, handlers[key])
        }
        delete delegator.sdDelegationHandlers
    }
```

---

#### separate binding into its own file [832e975](https://github.com/vuejs/vue/commit/832e97588d0a600d2a29ff406ebf1648341e9043)

clean up binding [60e246e](https://github.com/vuejs/vue/commit/60e246eedbd8d0a2be44a5ef2274f506d322b931)

把负责绑定和数组监听的部分拆分到 `binding.js`，部分变量重命名

---

#### sd-if [60a3e46](https://github.com/vuejs/vue/commit/60a3e46fbb8b1f86f29c080f50ef3329e6be40c7)

加入`sd-if`指令。

```js
'if': {
    bind: function () {
        this.parent = this.el.parentNode
        this.ref = this.el.nextSibling
    },
    update: function (value) {
        if (!value) {
            if (this.el.parentNode) {
                this.parent.removeChild(this.el)
            }
        } else {
            if (!this.el.parentNode) {
                // insertBefore时需要考虑原始位置。
                this.parent.insertBefore(this.el, this.ref)
            }
        }
    }
}
```
这里`insertBefore`是需要维护**原来的位置**的，所以要维护一个`参考节点`即`nextSibling`

---

#### sd-style [646b790](https://github.com/vuejs/vue/commit/646b790863110508ec0f074d75d262b4159384fb)

`style`指令。

现在的写法，以`borderColor`为例：

> dom :  sd-style="borderColor"
> scope: borderColor = '#000'

是以`-`做分隔符，比如`borderColor`这个style，写成`border-color`

额，一个初级版本吧。

---

#### remove unused [62a7ebe](https://github.com/vuejs/vue/commit/62a7ebe7db8306f9041880ff52508b23ae0c1d28)

嗯，把各种`matchSelector`干掉了 

---

#### remove redundant dependencies for computed properties [76ee306](https://github.com/vuejs/vue/commit/76ee306bdfa60e19cc3d3981547b64a68af4d2f4)

原先的依赖解析提取出来, 单独作为流程的一步，这样清晰多了。

```js
this._computed.forEach(parseDeps)
this._computed.forEach(injectDeps)
```

同时，原来的依赖解析有`重复依赖`的问题，每次`get`都会去push，要修一下

```js
function parseDeps (binding) {
    depsObserver.on('get', function (dep) {
         if (!dep.dependents) {
             dep.dependents = []
             }
        dep.dependents.push.apply(dep.dependents, binding.instances)
    })
}
```

放在了binding.dependencies里面：

```js

/*
 *  Auto-extract the dependencies of a computed property
 *  by recording the getters triggered when evaluating it.
 *
 *  However, the first pass will contain duplicate dependencies
 *  for computed properties. It is therefore necessary to do a
 *  second pass in injectDeps()
 */
function parseDeps (binding) {
    binding.dependencies = []
    depsObserver.on('get', function (dep) {
        binding.dependencies.push(dep)
    })
    binding.value.get()
    depsObserver.off('get')
}
```

修掉，原因作者说得很清楚了：

```js

/*
 *  The second pass of dependency extraction.
 *  Only include dependencies that don't have dependencies themselves.
 */
function injectDeps (binding) {
    binding.dependencies.forEach(function (dep) {
        if (!dep.dependencies || !dep.dependencies.length) {
            dep.dependents.push.apply(dep.dependents, binding.instances)
        }
    })
}


```

---

#### html and attr directives [f9077cf](https://github.com/vuejs/vue/commit/f9077cfa6afe39bfc6a7ffc0f81074ac95221b4b)

加了`attr`和`html`的指令

---

#### text parser started, features, optional oneway binding for input [7fd557c](https://github.com/vuejs/vue/commit/7fd557ccc8d8636c207229d289a8c45d2445a176)

`todo.md` 中可以看到在思考`组件化`的问题 

开始搞`text-parser`, 原来是`sd-text`

现在使用`{{}}`语法 

单次绑定需要加一个`-oneway`标记，如`sd-checked-oneway`

内置指令对这个`oneway`会做处理。

#### text parser [5541b97](https://github.com/vuejs/vue/commit/5541b971ed4e0ce793a034057025bb7be117a12b)

在完善`text-parser`, parse完成后返回一个`token`数组 

还未到可用状态 

#### finish text parser [b5f0227](https://github.com/vuejs/vue/commit/b5f0227186c3d73b9eca26b05ccec93263a57d09)

OK, 是时候梳理一下`text-parser`

`api.bootstrap` --> `buildRegex`，即创建一个`{{(.+?)}}`的`Regexp`

初始化创建`Seed`实例 --> `_compileNode` --> 如果是`textNode`, 进入`_compileTextNode`

`textNode` 两种内置指令： `{{}}` 和 `sd-text`

如果match了`{{}}`的方式，就按`sd-text`的方式去`parse`再绑定:

```js
tokens.forEach(function (token) {
    // 这里是bug. createTextNode() 方法要求有1个参数
    var el = document.createTextNode()
    // token.key，如{{label}} --> label 
    if (token.key) {
        var directive = Directive.parse(config.prefix + '-text', token.key)
        if (directive) {
            directive.el = el
            seed._bind(directive)
        }
    } else { // else的情况，那就是纯文本，如 <i>text:{{label}}</i> 中的"text:"部分
        el.nodeValue = token
    }
    node.parentNode.insertBefore(el, node)
})
```

---

#### chinese readme [8c8a07d](https://github.com/vuejs/vue/commit/8c8a07dd7f808a19e3e06d3d707054ca911ddec5)

`directive.js`可以自定义directive的相关函数 

```js
var prop,
        definition = directives[directiveName]
    // mix in properties from the directive definition
    if (typeof definition === 'function') {
        this._update = definition
    } else {
        // 这里就是自定义的
        this._update = definition.update
        for (prop in definition) {
            if (prop !== 'update') {
                this[prop] = definition[prop]
            }
        }
    }

```
在`directive`目录下的内置指令，均有`bind`, `update`, `unbind`的钩子，
已经有现在的`custom directive`的味道了：

```js

module.exports = {
    ...
    bind: function () {
        ...
    },
    update: function (handler) {
        ...
    },
    unbind: function () {
        ...
    }
}
```


#### minor updates [0e91e50](https://github.com/vuejs/vue/commit/0e91e50df32fc86a9538265ad54c8b697e3f7313)

> 卖点：
> - gzip后5kb大小
> - 基于DOM的动态模版，精确到TextNode的DOM操作
> - 管道过滤函数 (filter piping)
> - 自定义绑定函数 (directive) 和过滤函数 (filter)
> - Model就是原生JS对象，不需要繁琐的get()或set()。操作对象自动更新DOM
> - 自动抓取需要计算的属性 (computed properties) 的依赖
> - 在数组重复的元素上添加listener的时候自动代理事件 (event delegation)
> - 基于Component，遵循CommonJS模块标准，也可独立使用
> - 移除时自动解绑所有listener

另外做了一些cleanup.


---

#### move defineProperty() into Binding, add setter for computed property [52645e3](https://github.com/vuejs/vue/commit/52645e33bfc1d44ff93a8bace64f1ec74629deba)

对于`computed property` 加入了`setter`

```js
scope.allDone = {
    get: function () {
        return scope.remaining === 0
    },
    set: function (value) {
        scope.todos.forEach(function (todo) {
            todo.done = value
        })
        scope.remaining = value ? 0 : scope.total
    }
}
```
对于这个`allDone`, 加入了`set`; 有`set`属性的也说明是一个`computed property`

```js
/*
 *  Define getter/setter for this binding on scope
 */
Binding.prototype.defineAccessors = function (seed, key) {
    var self = this
    Object.defineProperty(seed.scope, key, {
        get: function () {
            ...
        },
        set: function (value) {
            // 这里不再使用emit('set')
            if (self.isComputed && self.value.set) {
                self.value.set(value)
            } else if (value !== self.value) {
                self.value = value
                self.update(value)
            }
        }
    })
}
```
---

#### separate deps-parser [e762cc7](https://github.com/vuejs/vue/commit/e762cc7ed56840cadcf0aa4d55a9da2bdf2272c3)

把`deps-parser`抽离，职责更清晰

---

#### clean up, add comments [7ad304e](https://github.com/vuejs/vue/commit/7ad304eb9de9546acc42775d25de4bd87fe070ce)

只在`sd-each`指令`update`时来定义`delegator`

不再单独定义`delegator`, 放在了`this.container.sd_dHandlers` (parentNode)中

---

#### parse key path in directive parser [c2faf1c](https://github.com/vuejs/vue/commit/c2faf1c1431c76055918dc92f1eb2d4835a2350d)

`key path`的作用显然就是用来做`nested property`的。。

```js
if (key.indexOf('.') > 0) {
    var path = key.split('.')
    key = path[path.length - 1]
    this.path = path.slice(0, -1)
}
```

举例：如`sd-text="a.b.c"`

```js
this.path = ['a', 'b']
```
。。。


---

#### nested properties [4126f41](https://github.com/vuejs/vue/commit/4126f41b08fe335224e0c197b0cb170fc5e3145d)


#### update nested props example [bf71151](https://github.com/vuejs/vue/commit/bf71151a1a78344c8ee43867320a3f0596b3011c)

`todo.md`: 已经在思考插件（或生态）的问题。。

> - sd-with
> - standarized way to reuse components (sd-component?)
> - plugins: seed-touch, seed-storage, seed-router

这两个commit一起看，搞定了`nested properties`

结合`examples/nested_props.html`做为示例测试：

```html
<h1>a.b.c : {{a.b.c}}</h1>
<h2>a.c : {{a.c}}</h2>
<h3>Computed property that concats the two: {{d}}</h3>
<button sd-on="click:one">one</button>
<script>
    var Seed = require('seed')
    Seed.controller('test', function (scope) {
        scope.one = function () {
            scope.a = {
                c: 1,
                b: {
                    c: 'one'
                }
            }
        }

        // computed properties also works!!!!
        scope.d = {get: function () {
            return (scope.a.b.c + scope.a.c) || ''
        }}
    })
    var app = Seed.bootstrap()
</script>

```

当key为`a.b.c`时，显然, 是需要对`c`做`getter`和`setter`，
原先的`defineAccessors`已经不适用， 
所以对于这个绑定表达式，需要判定当前的`path`有多长
递归地去做`defineAccessors`, 直到对`c`建立`getter`和`setter`

```js
Binding.prototype.defineAccessors = function (scope, path) {
    ...
    if (path.length === 1) {
        // 像就是a这样的expression,直接defineProperty
    } else {
        // 创建一个过渡的（中间的) subscope
        // 递归地defineAccessor, 直到path.length === 1

         // we are not there yet!!!
        // create an intermediate subscope
        // which also has its own getter/setters
        var subScope = scope[key]
        if (!subScope) {
            subScope = {}
            Object.defineProperty(scope, key, {
                get: function () {
                    return subScope
                },
                set: function (value) {
                    // when the subScope is given a new value,
                    // copy everything over to trigger the setters
                    for (var prop in value) {
                        subScope[prop] = value[prop]
                    }
                }
            })
        }
        this.defineAccessors(subScope, path.slice(1))
    }

}
```

增加了一个`getValue`函数, 也有这样的递归思想：

```js
function getValue (scope, path) {
    if (path.length === 1) return scope[path[0]]
    var i = 0
    /* jshint boss: true */
    while (scope[path[i]]) {
        scope = scope[path[i]]
        i++
    }
    return i === path.length ? scope : undefined
}
```

---

#### shorten some function names since they cant be mangled [fa45383](https://github.com/vuejs/vue/commit/fa453830ded8964013f3e3ed038fb208acfbbcdb)

重命名，压缩用

- defineAccessors --> def
- dependents --> subs
- denpendencies --> deps
- emitChange --> pub
- getScopeOwner --> trace

---


#### bootstrap returns single seep if that's the only one [f5995a5](https://github.com/vuejs/vue/commit/f5995a56416ab13d3336815104341a400280eb7b)

`sd-controller="a"` 出现多个时返回数组，仅出现一个返回`Seed`实例 

---

#### debug option [ad1cc3b](https://github.com/vuejs/vue/commit/ad1cc3be98d417b9a4b54951d86cd12e57c84a44)

增加`debug`模式，将一些常见报错`console.warn()`打印


---

#### restructure todomvc, add $watch/$unwatch [5200951](https://github.com/vuejs/vue/commit/5200951b0a3cd7e32f8b700762b653c89481c472)

增加一个`scope.js`  加入`$watch`和`unwatch`

`new Seed()`时会创建`Scope`实例：

```js
var scope = this.scope = new Scope(this, options)
```
`Scope`包含的内容：

```js
function Scope (seed, options) {
    this.$seed     = seed
    this.$el       = seed.el
    this.$index    = options.index
    this.$parent   = options.parentSeed && options.parentSeed.scope
    this.$watchers = {}
}
```

常用的一些自定义函数抛到了`utils.js`


---
#### 0.1.0 [5f5aa8f](https://github.com/vuejs/vue/commit/5f5aa8fb404ece37416abf405dd1a207e8997862)

打包完成， 0.1.0 done

梳理一下：

```js
├── binding.js // 绑定逻辑
├── config.js // 设置
├── deps-parser.js // computed property依赖解析
├── directive-parser.js // Expression解析器，同时负责创建Directive实例作为dom属性
├── directives // 内部指令
│   ├── each.js // sd-each数组指令，包括一系列mutationHandlers
│   ├── index.js // 内部指令入口点及一系列常用的directive
│   └── on.js // 事件监听
├── filters.js // filter处理
├── main.js  // 入口点
├── scope.js // Scope class,包含了seed实例，dom,$watchers,$parent等信息
├── seed.js // main ViewModel 
├── text-parser.js // textNode parser
└── utils.js // utols, 一些公用函数
```

里面最复杂的部分应该是：

> - `_compileNode` 节点解析
> - `computed properties`自动计算 

**过一下`_compileNode`:**

根据节点类型做不同处理:

- `textNode` --> 解析纯文本节点`_compileTextNode(node)`
- `nested controllers` --> 直接创建新实例`new Seed()`, parent是当前Seed
- `normal node` --> 
    - 【 遍历`attributes` --> `DirectiveParser.parse(attr.name, exp)`得到一个`Directive`新实例(d) --> 创建或使用现有`binding`， 把d添加到正确的`binding.instances`上 `SeedProto._bind(d)` 】
    -  --> 如果有`childNode`则递归


**过一下`computed properties`的实现：**

- 核心是利用`Emiiter`的观察者模式，在`scope`上定义的属性，当其存在`getter`时, 收集依赖
- 依赖只能依赖没有依赖的依赖, 举例：
    
```js
// 说明remaining依赖于total和completed
scope.remaining = {
    get: function () {return scope.total - scope.completed}
}

// 说明total依赖于todos
scope.total = {
    get: function () {return scope.todos.length}
}

```
这样，`remaining` 经过`deps-parser`后，实际依赖是 `todos` 和 `remaining`, `total`由于有其他依赖，被干掉 

依赖改变时，通过接收到`get`事件，调用自己的`get`来更新。

---

PS：改了`grunt`打包之后，整个`seed`在一个立即执行function里面，原来的Demo应该都不能用了。。。

```bash
Uncaught ReferenceError: require is not defined
```
这当然了，因为都没有暴露到`window`里面去嘛......

把第一行和最后一行注掉即可


## 0.2.0 （branch 0.10）

#### computed properties now have access to context scope and element [8eedfea](https://github.com/vuejs/vue/commit/8eedfeacf9f44d0ae33118816c2b352a2d1b420f)

让`computed properties` 有访问`dom`和`scope`的能力

值得关注的改动点是`src/deps-parser.js`

```js
function catchDeps (binding) {
    ...
    binding.value.get({
        scope: createDummyScope(binding.value.get),
        el: dummyEl
    })
    ...
}
```

这个`createDummyScope` 是在搞啥呢？

看这段注释：

```
/*
 *  We need to invoke each binding's getter for dependency parsing,
 *  but we don't know what sub-scope properties the user might try
 *  to access in that getter. To avoid thowing an error or forcing
 *  the user to guard against an undefined argument, we staticly
 *  analyze the function to extract any possible nested properties
 *  the user expects the target scope to possess. They are all assigned
 *  a noop function so they can be invoked with no real harm.
 */
```
首先，如果要让`computed property`访问`scope`和`dom`，就得在`get`时传入参数。：
举例，这个参数是`e`

```
e = {$el: xxx, $scope:xxx}
```

假设用户这么去写

```js
scope.a = {}
scope.compute_a = {
    get: function (e) {
      return  e.scope.a.b.c + 1
    }
}

```
那这样肯定是会报错的。

> 但是却不能约束用户不去这么写。比如我是用户，我需要一个`promise`回来后，再赋值`scope.a = {b:{c:'hello'}}`，这时我也要求这个`compute_a` 能计算啊。这个要求合情理。

作者的实现是：（这里的e也可以是其他啦，一样的）

- 把`get`函数转成字符串，匹配`function (e) {xxx}`部分
- 取出`xxx`部分，找匹配`e.scope.a.b.c`
- 解析成path, ['a', 'b', 'c']
- 看看a,b,c都有没有值咯，有值传值，没值将值赋成空函数`noop`(`function(){}`)

如果a,b,c都没值的话，最后解析的结果：

```
e.scope.a = function(){}
e.scope.a.b = function(){}
e.scope.a.b.c = function(){}
```
---

#### avoid duplicate context dependencies [a5727bd](https://github.com/vuejs/vue/commit/a5727bd4b37caffe1f8f0ae0fd615154c30e8e5d)

`catchDeps` --> `parseContextDependency`

避免`deps`重复的情况。

---

#### use for loops instead of forEach whenever possible [2d448ea](https://github.com/vuejs/vue/commit/2d448ea5e5d19b231aa8a525248aff11356c9936)

`forEach` 都改写成普通的`for`和`while`循环，应该是性能方面的考虑。

---

#### add pluralize filter [9546965](https://github.com/vuejs/vue/commit/9546965ae0d0c578227394a2d5c64c6c44b461fd)

【复数】形式的`filter`, 例如数量为1时显示`item`, 大于1时是`items` 
 这个好像必然是要干掉的 -- 复数后缀是`es`的咋办 = = 

--- 

#### separate storage in todos example, expose some utils methods [cfc27d8](https://github.com/vuejs/vue/commit/cfc27d89f1ff841ec9edd0c27d34b5111b098d4c)

把`todos`里面的内容存在`localStorage`

把下列函数放在了新建文件 `utils.js`

- typeOf
- dump
- serialize
- getNestedValue
- watchArray

---

#### createTextNode in FF requires argument [8a004e7](https://github.com/vuejs/vue/commit/8a004e787598bc4a146e6045d97301dd4c2f5123)

`document.createTextNode()` ==> `document.createTextNode('')`
 
作者终于把这个bug修掉了... 之前的commit都要手动去改这里。。

奇怪的是，难道作者当时在用IE测试吗。。。
 
查了一下[资料](https://www.w3.org/TR/DOM-Level-3-Core/core.html#ID-1975348127)，包括[Mozilla Developer](https://developer.mozilla.org/en-US/docs/Web/API/Document/createTextNode)都似乎并没有说参数是必选的。

IE全系可以不带参数。`Chrome`和`FF`是必带的。各浏览器实现不同吧。。

---

#### stricter check for computed properties [5bfb4b5](https://github.com/vuejs/vue/commit/5bfb4b5ef2919c23cb0f9208081980d70f7be15f)


```js
if (type === 'Object') {
    // 原来的判断 
    // if (value.get || value.set) { self.isComputed = true}
    if (value.get) {
        var l = Object.keys(value).length
        if (l === 1 || (l === 2 && value.set)) {
            self.isComputed = true // computed property
        }
    }
}
```

规定了必须有`get` 才能是`computed property`

--- 

#### add Seed.broadcast() and scope.$on() [2658486](https://github.com/vuejs/vue/commit/2658486bcc096ac8edfd46655115cb5373935f01)

利用`Emitter` 实现事件系统, 绑定到`seed._listener`上 

`on`和`off`来绑定和解绑

---

#### put properties on the scope prototype! [cc64365](https://github.com/vuejs/vue/commit/cc6436532376938c6a6be7a020ce2461dbb500e1)

把`computed properties`的`get`和`set`搞到了`scope`的原型上 

```js
if (l === 1 || (l === 2 && value.set)) {
    self.isComputed = true // computed property
    value.get = value.get.bind(self.scope)
    if (value.set) value.set = value.set.bind(self.scope)
}
```

`api.controller`中引入了一个`ExtendedScope`, 注释如下


> // create a subclass of Scope that has the extension methods mixed-in

```js
var ExtendedScope = function () {
        Scope.apply(this, arguments)
    }
    var p = ExtendedScope.prototype = Object.create(Scope.prototype)
    p.constructor = ExtendedScope
    for (var prop in properties) {
        if (prop !== 'init') {
            p[prop] = properties[prop]
        }
    }
```

典型的`原型继承`， 这说明`controller`实际上就是`scope`的子类

`seed.js`中；如果有`ExtendedScopeConstructor`则使用，否则用原生的

>    // create the scope object
>
>    // if the controller has an extended scope contructor, use it;
>
>    // otherwise, use the original scope constructor.

**为什么要引入这个东西，要思考下。。这里存疑**

---

#### rename internal, optimize memory usage for GC [d78df31](https://github.com/vuejs/vue/commit/d78df3101711ee85b4c3ebf5d488a0d4e9e89ad6)

#### really fix GC [a85453c](https://github.com/vuejs/vue/commit/a85453cc89a13c239fccad1bb7b276c4fe473e0a)

- `seed.js` 改名为 `complier.js`
- `scope.js` 改名为 `viewmodel.js`

这个命名比原来的赞多了...

`scope -> vm` 容易让人想到`angular` 的 `controller as`...

干掉了好多`self`换成了`this`感觉清爽许多，当然一些地方需要配合`Function.prototype.bind`

加入了`BindingProto.unbind` 

GC:

```js
this.vm = this.compiler = this.pubs = this.subs = this.instances = this.deps = null
```

---

#### binding should only have access to compiler [f071f87](https://github.com/vuejs/vue/commit/f071f87bc7f0931e0f6382aefb98c4e01d81505a)

`binding`直接使用`complier`中的`vm` 而不是自己定义一个；

---

#### new api WIP [761b643](https://github.com/vuejs/vue/commit/761b643baad247394f35877e7cc25624cd9e92e3)

开发模式大改变，隐隐有现在的味道了：

```js
Seed.ViewModel.extend({
    template: 'xxx', //optional
    el: 'xxx', //optional
    initialize: function () {...},
    properties: {
        // 属性
        total: {get: function (){...}},
        // 方法。
        addTodo: function () {...}
    },

})
```

对比原来：

```js
Seed.controller('todos', {
    // initializer, reserved
    init: function () {
       ...
    },
    // computed properties ----------------------------------------------------
    total: {get: function () {
        ...
    }},
    // event handlers ---------------------------------------------------------
    addTodo: function () {
        ...
    },
})

Seed.bootstrap()

```

对比原来的好处：

- 结构更清晰
- 开发者理解更加简单：不用去写`sd-controller`, 不用去调用`Seed.bootstrap()`
- 不用去理解`controller`和`scope`这些概念名词，传`el`即可
- 更易复用和组件化，整个`viewmodel`即是一个`object`

`MVVM` 模式下：

> 写html(绑定`properties`和`methods`) -> 写viewModel(定义`properties`和`methods`) 
>
> `properties` 改变时通知html改变视图； html改变时触发`properties`改变或触发`method`

这么做的代价并不高，但是带来的开发体验感觉是颠覆式的~ 

俺个人就很不喜欢`controller`和`scope`这两个词儿 = =

---


#### working, but with memory issue; [c98c8a6](https://github.com/vuejs/vue/commit/c98c8a68cb2eee3acfc22e19b86ee9ba3f899a63)

> each.js

```js
var config = require('../config'),
    ViewModel // lazy def to avoid circular dependency

buildItem: function (ref, data, index) {
    ...
    ViewModel = ViewModel || require('../viewmodel')
    ...
},

```

`ViewModel.js` 需要require  `compiler.js`

`compiler.js` 需要 require `directive-parser`

`directive-parser` 需要 require `directives/index.js`

`directives/index.js` 需要 require `each.js`

`each.js`又需要require`ViewModel.js`....

绕了半天就形成了一个`circular dependency`

--- 

#### new api fixed [dddb255](https://github.com/vuejs/vue/commit/dddb2550740d1e991643a95628d34c8c38ae4f54)

#### 0.2.0 [253c26d](https://github.com/vuejs/vue/commit/253c26db1a73cb5f635c4814b07e3fa7f20e64de)

少量修正，编译0.2.0

---

### 总结一下0.1.0到0.2.0的升级改进：

- computed properties 可以访问 `scope(viewmodel)` 和 `dom`
- 基于`Emitter`引入了事件`pub/sub`系统 
- 引入了`viewmodel`概念，开发范式接近现在的`Vue`
- 性能优化类：循环优化，GC等

---

#### optimize array watch method hijacking [a104afb](https://github.com/vuejs/vue/commit/a104afb472e62a5b52e0a6f6d21aa2d920b1959c)

把Array相关的Mutation方法提取出来了 

---

#### fix dump [174a19f](https://github.com/vuejs/vue/commit/174a19f5ec42130359a86e18194082d60121f887)

照例看`todo.md`

- ability to register a ViewModel so it can be auto-compiled as a nested vm
- literals, function arguments, simple logic comparisons
- let user specify which keys are data/state (i.e. available in $dump())

- 声明一个 `viewmodel` 时自动编译啦 
- 字面量，函数参数，简单的逻辑对比
- 用户指定哪些是`data`哪些是`state` (这的`state`是类react的概念？)

学英文:  `contextual binding` 结合上下文的

```js

var ARGS_RE = /^function\s*?\((.+?)[\),]/,

args = str.match(ARGS_RE)
if (!args) return null
binding.isContextual = true
```


这里意味着：

- 这个RegExp匹配的是形如 `function (abc) ` 这样 **带参数的**
- 当`computed property` 需要用到当前上下文的时候， `isContextual = true `
- 这种情况下，`dump()`时会跳过

举例：
```
// dynamic context computed property using info from target viewmodel
filterSelected: {get: function (ctx) {
        return this.filter === ctx.el.textContent.toLowerCase()
    }},
```

---

#### support nested VMs and update examples to use new API [bf01a14](https://github.com/vuejs/vue/commit/bf01a14629e0b11ab33a5e20702eba94e281cbe8)

`todo.md` 已经在考虑生态问题。。。

- sd-with?
- plugins
    - seed-touch (e.g. sd-drag="onDrag" sd-swipe="onSwipe")
    - seed-storage (RESTful sync)
    - seed-router (express style)
    - sd-validation (e.g. sd-validate="type:email, max:100, required:true")

为当前的`viewmodel`模式支持了 `nested VMs` 如 `a.b.c`

---

#### simply api; [6f0eca4](https://github.com/vuejs/vue/commit/6f0eca4bf6447c5b2c570322e56135a011585429)

**开发者体验至上的理念**，赞~ 


--- 

#### todos [0afd700](https://github.com/vuejs/vue/commit/0afd7005912a8abbb020496e5511d88c1c1c97e2)

- add a few util methods, e.g. extend, inherits
- fix architecture: objects as single source of truth...
- prototypal scope/binding inheritance (Object.create)


- 加些`util方法`
- 改架构，对象作为单一数据源~~
- 原型继承~

--- 

#### watcher [e028262](https://github.com/vuejs/vue/commit/e028262582e796d5d8bbc5c135e16a92279f7a10)

#### external observe [b89092b](https://github.com/vuejs/vue/commit/b89092b31752c3a824f1e823603d65e2dd9612c6)


`Array` => `arrayMutators` 

`Object` => `Emitter`

通过`Object.defineProperty`定义了几个`enumerable:false(不可枚举)` 、 `configurable:false(不可删改)` 的变量：

- `__path__` 路径信息如`a.b.c` 
- `__value__` 即值的副本
- `__observer__` 用于事件emit

最后，这个`watcher`的作用就是： 

当变量`get|set|mutate`时，触发事件，结合现在的`watch`，很好理解 

另外，`Array`在`push`进来一个新数据的时候要添加`watcher`，作者的思考是否是放在`each`里面搞，不单独放到`watcher.js`


---

#### wip [84538d6](https://github.com/vuejs/vue/commit/84538d6daefd0e1fdc6ec5efbeb377172adec556)

小重构

`Object.defineProperty` 相关被从`binding.js`移入`complier.js`

`binding.js` 做的事情现在只有： 

- update
- refresh
- unbind
- pub 

做的事情更纯粹，即在数据变化时做一个「执行者」角色

`watchArray`相关被移出`util.js`做为`observer.js`的一部分，这样从概念上感觉更职责清晰

arrayWatcher 只在observer时使用，并不是一个太公用的方法。

---

#### make dep parsing work [ea792ba](https://github.com/vuejs/vue/commit/ea792ba6bb8d43c79d47e1256e04dbf7af53ef91)

`computed properties（依赖get function中的成员）`和已经有`__observer__（依赖下层级成员）`的属性们，你们已经不干净了！不纯洁了！

所以不需要`emit`了。


```js
Object.defineProperty(this.vm, key, {
    enumerable: true,
    get: function () {
        if (!binding.isComputed && !binding.value.__observer__) {
            // only emit non-computed, non-observed values
            // because these are the cleanest dependencies
            compiler.observer.emit('get', key)
        }
        return binding.isComputed
            ? binding.value.get({
                el: compiler.el,
                vm: compiler.vm
            })
            : binding.value
    },
    set: function (value) {
        if (binding.isComputed) {
            if (binding.value.set) {
                binding.value.set(value)
            }
        } else if (value !== binding.value) {
            compiler.observer.emit('set', key, value)
            observe(value, key, compiler.observer)
        }
    }
})
})
```

---

#### add asyn compilation ($wait/$ready) [beded7d](https://github.com/vuejs/vue/commit/beded7d779a57ab6de42f89a69f4410ccdf64080)

为异步加载的数据引入`$wait`和`$ready`

如果调用了`vm.$wait()` 那么暂时不进行`compile`

在用户执行`vm.$ready()` 的时候，使用`Emitter`派发事件，进行`compile`

---

#### todo [0f56b85](https://github.com/vuejs/vue/commit/0f56b858887bf79799cd1068f8b4907bb84356c8)

- $watch / $unwatch & $destroy(now much easier)
- add a few util methods, e.g. extend, inherits

> 最后，这个`watcher`的作用就是： 
>
> 当变量`get|set|mutate`时，触发事件，结合现在的`watch`，很好理解 


嗯，如前面的笔记，显然`watcher`是为 `$watch,$unwatch,$destroy` 服务的

---

#### unobserve when setting new values [caed31f](https://github.com/vuejs/vue/commit/caed31fd02ea9f9ab512bb1f2867ed1a52d01a84)

作者在这个commit里面做了很多替换，多次访问属性的全搞出来，细微的性能优化也要优化~追求极致啊

```js
if (this.contextBindings.length) this.bindContexts(this.contextBindings)
```

to:


```
var contextBindings = this.contextBindings

if (contextBindings.length) thisbindContexts(contextBindings)

```

当value改变时，这些old value已经不需要再watch了。emit get/set/mutate全置nullGC之~~

```js
set: function (value) {
    ...
    else if (value !== binding.value) {
        // unwatch the old value!
        Observer.unobserve(binding.value, key, compiler.observer)
        // now watch the new one instead
        Observer.observe(value, key, compiler.observer)
        binding.value = value
        compiler.observer.emit('set', key, value)
    }
}
```

---

#### templates [d2779aa](https://github.com/vuejs/vue/commit/d2779aad828f94d636bdbef1579cba161bfdd082)

额咋又把`api.bootstrap`搞回来了。。

要支持`template`这玩意儿，作者当前想法是在`script`标签里面搞：

```
<script type="text/sd-template" sd-template-id="test">
    <p>{{hi}}!</p>
</script>
```
拿到`innerHTML`, 给它一个id: `test`, 存起来，取的时候用这个id取

搞完之后把这个script干掉 

---

#### wip [bc84435](https://github.com/vuejs/vue/commit/bc84435133126e1f080571d624af11beb24171b5)

`VMProto.$set` 用于处理在`value change`时类似`a.b.c`的赋值；

如： `sd-checked = "a.b.c"`

这里实现很优雅，学习一下

如果没有"." 那么 `vm.a = value`;

如果有那么一直往下搞。。 最后是 `vm.a.b.c.xxx = value`

```js
VMProto.$set = function (key, value) {
    var path = key.split('.'),
        level = 0,
        target = this
    while (level < path.length - 1) {
        target = target[path[level]]
        level++
    }
    target[path[level]] = value
}
```
---

#### trying to fix nested props [6ec70eb](https://github.com/vuejs/vue/commit/6ec70eb4a4811c6e13bda378a5938b0f7d307b0a)

- auto add .length binding for Arrays

嗯这个是必要的。

src/deps-parser.js：

`filterDeps` 完全干掉：由于emit的时候跳过computed property, 这玩意儿已经木有用了嘛。。

src/compiler.js：

`createBinding()`时维护了一个数组`observables`，用于收集**非computed**的vm属性

遍历完vm后，observables也就收集完了，此时再进行`observe`


```js
for (key in vm) {
    if (vm.hasOwnProperty(key) &&
        key.charAt(0) !== '$' &&
        !this.bindings[key])
    {
        this.createBinding(key)
    }
}

...

if (type === 'Object') {
    if (value.get) {// computed property
        ...
    } else {
        // observe objects later, becase there might be more keys
        // to be added to it
        this.observables.push(binding)
    }
} 

...

// define root keys
var i = observables.length, binding
while (i--) {
    binding = observables[i]
    Observer.observe(binding.value, binding.key, this.observer)
}

```
---

#### fix root key setter trigger order [1dd9c0e](https://github.com/vuejs/vue/commit/1dd9c0edcc8daf5f1d898f8d3bacb7b1ae9b602e)

学英文。。忽略我俗俗的翻译水平

/*
 *  Sometimes when a binding is found in the template, the value might
 *  have not been set on the VM yet. To ensure computed properties and
 *  dependency extraction can work, we have to create a dummy value for
 *  any given path.
 */

有时在模板中找到一个绑定变量时，值还没设哪（比如异步），那么为了让「计算属性」和「依赖提取」好使，需要为所有给定路径创建一个“虚拟变量”

举个例子吧：比如定义一个`sd-checked`但又没有初始化`a.b`, 通过一个异步请求搞到并赋值

```js
<input type="checkbox" sd-checked="a.b">

vm.checked = {get: function (){return !!a.b}}
vm.a = {} 

fetch(url).then(function(res) {
  if (res.ok) {
    res.json().then(function(data) {
      vm.a.b = true
    });
  }
});

```
那么在`CompilerProto.ensurePath`中会先给a.b搞成`undefined`

---

#### allow multiple VMs sharing one data object [75a4850](https://github.com/vuejs/vue/commit/75a4850afb4e5c598bbe14cbea6e594ac48da07d)

新场景 ： 一份data被2个（或以上）vm使用 

是判断有没有`__observer__` , 有的话再触发一次

---

#### small cleanups [0a45bb8](https://github.com/vuejs/vue/commit/0a45bb813f9657c4409ee5a597a6125a09c65141)

经常性地做一些cleanup，包括todo.md的编写对代码质量提升很有好处。
向尤大学习科学的coding素养！

---

#### directly use array length [77ce1fc](https://github.com/vuejs/vue/commit/77ce1fc35552c2f85ab592b24773bf96e5a05bbd)

这个实现有点暴力了 = =

咋搞出个`this.array`专门做array length处理。。 画风突然变了的赶脚。。。。

--- 

#### fix each.js for new architecture [c6c5fdb](https://github.com/vuejs/vue/commit/c6c5fdb3d8ceadc4602b24dd9f22d88393b39cc5)

果然果断把上面的实现干掉了。。

Array就是observables, 直接emitSet即可 

each.js里面按照vm的模式搞了一遍 

---

#### todo [1979aea](https://github.com/vuejs/vue/commit/1979aeab4a4aaf6bee2fa696145c99ab5ce115ed)

- consult https://github.com/RubyLouvre/avalon/issues/11 to allow simple expressions in directives.

尤大当时居然在看`avalon`..

这里面的实现搞进去看到了[artTemplate](https://github.com/aui/artTemplate)

模板引擎搞起来没个边啊（parser? 调试? 性能?。。。）

对于MVVM框架这个的「边际」是一个很微妙的事情，做多了有些喧宾夺主，做少了也不行

我理解支持常见运算符就能满足大部分需求了：

- 比较运算符,如"> === <"
- 算术运算符，如`%`
- 逻辑运算符，如`&& ! ||`
- 条件运算符(即三目)

比如位运算符，一些关键字像delete, typeof, void，instanceof这些，感觉支持了也没有什么卵用...没啥使用率啊

后面看作者是怎么思考这个边界的...

---


