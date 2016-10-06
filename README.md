# Vue learing notes

My note of learning Vue.js


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

PS：每个commit的标题，当然不能复制粘贴手写... chrome snippet来一套

```js
(function () {
    let href = window.location.href;
    let key = window.location.pathname.match(/(commit\/)[\w]+/g)[0].split('/')[1];
    let shortKey = key.slice(0,7);
    let commitTitle = document.querySelector('.commit-title').innerText;
    let $commitDesc = document.querySelector('.commit-desc');
    let commitDesc = $commitDesc ? $commitDesc.innerText : '';
    let commitText = (commitTitle + commitDesc).replace(/……/g, '');
    let result = `#### ${commitText} [${shortKey}](${href})`;
    console.log(result);    
})();
```


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

## 0.1.0 （branch 0.10）done


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

## 0.2.0 （branch 0.10）done

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

#### fix init value for Directives [9c4b62f](https://github.com/vuejs/vue/commit/9c4b62fd508cb1d5152aa014054d54ad4a236ecc)

```js
/*
 *  called when a new value is set 
 *  for computed properties, this will only be called once
 *  during initialization.
 */
DirProto.update = function (value, init) {
    if (!init && value === this.value) return
    this.value = value
    this.apply(value)
}
```
加了个参数`init`，但是好像没看到哪里有这么调的？这有啥用？

**存疑**

---
#### add utils.extend [e1ce623](https://github.com/vuejs/vue/commit/e1ce623035957b4d11a80920d49249ff4f65013c)

```js
    extend: function (obj, ext) {
         for (var key in ext) {
             obj[key] = ext[key]
         }
     },
```
---

#### expression parsing [9d0d211](https://github.com/vuejs/vue/commit/9d0d2114f8be447cd2aa0032d14b9b85278bc596)

`src/exp-parser.js` 加入了expression parsing

参考的`artTemplate` 实现 

> 算法采用过滤的思路实现：
>
> 1. 删除注释、字符串、方法名，这一步是为了排除干扰
> 2. 删除可组成变量的字符，只保留字母、数字、美元符号与下划线，并进行分组
> 3. 删除分组中的 js 关键字与保留字成员
> 4. 删除分组中的数字成员


```js

    REMOVE_RE   = /\/\*(?:.|\n)*?\*\/|\/\/[^\n]*\n|\/\/[^\n]*$|'[^']*'|"[^"]*"|[\s\t\n]*\.[\s\t\n]*[$\w\.]+/g,
```

最长的这个RE拆开就没啥啦

\/\*(?:.|\n)*?\*\/|\/\/[^\n]*\n  -> 多行注释

\/\/[^\n]*$ -> 单行注释

'[^']*'|"[^"]*" -> 字符串

[\s\t\n]*\.[\s\t\n]*[$\w\.]+  排除a.b.c干扰（.前面可能有\s\t\n）

举例：

```
<p sd-text="one + ' ' + two + '!'"></p>
```
getVariables(expresion) => ['one', 'two']

搞出函数表达式args => 

args = ['two=this.$get('two')', 'one=this.$get('one')']

args拼装 => `var two=this.$get('two'), one=this.$get('one');return one + ' ' + two + '!'`

new Function(args), 得到Expression的值

这里最后把这个`Expression`当作一个`computed`成员来处理，思路很清楚

---

#### 0.3.0 [d4beb35](https://github.com/vuejs/vue/commit/d4beb35b68d3fe067cacb2627567703b0efe00bb)

0.3.0版本

## 0.3.0 （branch 0.10） done 
    
---

#### make it cleaner [b379274](https://github.com/vuejs/vue/commit/b379274b31c8f94e45e5d407517745895fe1f998)

果然大神都有的特点：强迫症，洁癖，完美主义 

---

#### $watch/$unwatch [874afe2](https://github.com/vuejs/vue/commit/874afe2389bbb4c4fbf9b55429c0935b0a9724d0)

$watch和$unwatch直接代理到了observer上, 监听change事件~ 

干净利落 

---

#### comments [d7f753e](https://github.com/vuejs/vue/commit/d7f753eff59a7fe1830e5b13bdcd44e45cc12209)

各种没注释的都加了很清晰的注释...

---

#### get ready for tests [c6903e0](https://github.com/vuejs/vue/commit/c6903e0074c8a2b74ea2ba5f6037ea0ffdb1dda4)

这个大阵仗，要开始写测试了。。。

vue的测试覆盖率是恐怖的**100%**...

---

#### 0.3.2 - make it actually work for Browserify [498778e](https://github.com/vuejs/vue/commit/498778ec23e210be8affe331c9d86071ec79d461)

赶时髦兼容`browserify`了... 对于13年来说，browserify的确是非常时髦的。。。

---

#### readme, todos [87f603f](https://github.com/vuejs/vue/commit/87f603f3723d386fefa9c2aac754370142835d4d)

- ability to create custom tags

组件化的第一步！

---

#### unit tests for binding.js [df21257](https://github.com/vuejs/vue/commit/df212574fd2cc5009769d3aed8c2e76acd7aeb0f)

#### unit tests for directive.js [b23c790](https://github.com/vuejs/vue/commit/b23c790fbe22c8ccae98a7b7694fa0bb3d938838)

#### fix directive test [5685f68](https://github.com/vuejs/vue/commit/5685f6853af70f4a221e041b8aff5751608a9037)

#### use strictEqual [7fdb1b2](https://github.com/vuejs/vue/commit/7fdb1b25a7ea4c121f4c254fa593c6cb0c72301b)

单元测试,撸码如有神

---

#### decouple compiler and exp-parser [1ef6571](https://github.com/vuejs/vue/commit/1ef6571c940f5f6e13f095b768d1550fb94746a4)

需要从`exp-parser`中获得parse后的变量来创建binding

---

#### unit test for Expression Parser [d80b5ff](https://github.com/vuejs/vue/commit/d80b5ffef6a554fa79dcd6954e9125c43aa12255)

#### unit test for Dependency Parser's internal methods [2433a3a](https://github.com/vuejs/vue/commit/2433a3a69c44edd60e286c805ca6a923c24cbaab)

#### unit test for TextParser [1c85e86](https://github.com/vuejs/vue/commit/1c85e86297cbdecedf754444bd5a9bc41f88048c)

...

#### make all unit tests run in real browsers [6bc19e6](https://github.com/vuejs/vue/commit/6bc19e6e6674b3ecc5d3644e19ba45803ed34181)

暂时先mark吧...这写测试的速度...已跪

---


#### use __proto__ interception for array methods [75dcb03](https://github.com/vuejs/vue/commit/75dcb03eec1ae6fc17d9c37ffa64272197433a04)

拦截了`__proto__` 干掉了原来的`arrayMutators`

but...`__proto__`作为原型访问器的特性，IE11以下是不支持的，确定要这样吗。。

```js
var ArrayProxy = Object.create(Array.prototype)
ArrayProxy.remove = function () {...}
ArrayProxy.replace = function () {...}
ArrayProxy.mutateFilter = function () {...}


arr.__proto__ = ArrayProxy


```
---

#### simplify template API [c94ff6b](https://github.com/vuejs/vue/commit/c94ff6b03355829d4ae15f5f0352328ceea3d5b4)

```js
// determine el
var el  = typeof options.el === 'string'
    ? document.querySelector(options.el)
    : options.el
        ? options.el
        : options.template
            ? utils.makeTemplateNode(options)
            : vm.templateNode
                ? vm.templateNode.cloneNode(true)
                : null
```

原来的方式是传这个ID：

```html
<script type="text/sd-template" sd-template-id="test"></script>
```

如果有options.el且是string => 搞这个dom

如果有options.el是dom => 搞这个dom

如果没有options.el, 有options.template(string): 通过utils.makeTemplateNode创建一个dom

没有options.template，有vm.templateNode => 复制

都没有 = = 那就木办法了 

**这样做的好处：template可以任意传string, 原来的约束较大。**

---

#### fix observer mechanism [5ff47a8](https://github.com/vuejs/vue/commit/5ff47a83dde64f8f6a67e4543e3f522338d5b1ae)

之前通过emitSet递归,对于`nested properties`（如a.b.c）有可能有重复emit的现象

现在observe寄存到了各级属性里，直接emit即可

```js
if (alreadyConverted) {
                emitSet(obj, ob, rawPath)
            }
```

```js
function emitSet (obj, observer) {
    if (typeOf(obj) === 'Array') {
        observer.emit('set', 'length', obj.length)
    } else {
        emit(obj.__values__)
    }
    function emit (values, path) {
        var val
        path = path ? path + '.' : ''
        for (var key in values) {
            val = values[key]
            observer.emit('set', path + key, val)
            if (typeOf(val) === 'Object') {
                emit(val, key)
            }
        }
    }
}
```
---

#### template changes again + allow further extend() for VMs [949e6e1](https://github.com/vuejs/vue/commit/949e6e1521003057c10a18d59211a5bc016ef6ec)

抽离出来一个`src/template.js`

`template`还是恢复到允许定义一个`template name`

---

#### sd-if and minor fixes [4209e27](https://github.com/vuejs/vue/commit/4209e272607278768a8c403b5b9b1a8fb78c57c1)

`todo.md`

> - sd-partial
> - transition effects
> - component examples

完成了`sd-if`

---

#### implement new API per spec [4003fe2](https://github.com/vuejs/vue/commit/4003fe2b07cf53afabc3a28e2b0e75cc6fa31463)

#### new init/extend options API [8a94192](https://github.com/vuejs/vue/commit/8a9419241e3a51ccfeb58c6a81a0df807653dd52)

template => documentFragment

`src/template.js` 又干掉了


---

#### directive interface change [9297042](https://github.com/vuejs/vue/commit/9297042b2d8e8dae9e4d241156f2da71e66fe201)

template api的改动带来的directive改动 

```js
directive = Directive.parse(eachAttr, eachExp)
=>
directive = Directive.parse(eachAttr, eachExp, compiler, node)
```
---

#### private directives and filters [eef0dc6](https://github.com/vuejs/vue/commit/eef0dc6ad16680e9e86b376b972cf711d4a8557b)

用户的自定义directive和filter

---

#### ViewModel.extend() should also extend Object options [ec86b9f](https://github.com/vuejs/vue/commit/ec86b9faf94ee77827e182268c15e1396ae8e047)

ViewModel.extend()

```
function extend (options) {
    var ParentVM = this
    // inherit options
    options = inheritOptions(options, ParentVM.options, true)
    ...
}
```
`inheritOptions`是一个自定义的深拷贝,除去el和props:

- props不需要拷贝因为已经在prototype上了
- el只允许是一个`instance option`,所以也不需要拷贝 

---

#### $index for each items [2232cf2](https://github.com/vuejs/vue/commit/2232cf2885989bf225e72e2124d804597ebf715a)

`sd-each` 支持$index, 在有mutation时，需要updateIndexes()

---

#### array methods should be inenumerable [331f03b](https://github.com/vuejs/vue/commit/331f03b2fd5ec7dfd212c92ef181e82dd4ddb6f6)

嗯，array methods自己的扩展，并不是广义性的，用户也很难会去用。

---

#### detach container before batch DOM updates for sd-each [98d1108](https://github.com/vuejs/vue/commit/98d1108dd1851c706a358a9400a241c6e430debd)

#### avoid no parent detach [61e897e](https://github.com/vuejs/vue/commit/61e897ea8861355c84ca2a0090a7b8d65084c552)

批量更新

类似domfragment的思想，性能优化

---

#### implement $ event methods, optimize for minification [a21e890](https://github.com/vuejs/vue/commit/a21e8907d13a9df12ea89155626321779997f66c)

`src/compiler.js` 中template解析部分提取出来成为`compiler.setupElement(options)`

---

#### TODOs [81c705c](https://github.com/vuejs/vue/commit/81c705cd132038df5e02b42c57d1c1f07072080d)

> - change the name to one that no one has used before...
> - change the exposed var to a single one (capitalized constructor)
> - change the default prefix to a single letter (depending on new name)
> - change `sd-each` to `s-repeat`
> - put all API methods on Seed
> - add a method to observe additional data properties
> - add `lazy` option
> - add directive for all form elements, and make it consistent as `s-model`
> - add escape: {{{ things in here should not be parsed }}}
> - rename `props` option to `proto`
> - properties that start with `_` should also be ignored and used as a convention: `$` is library properties, `_` is user properties and non-prefixed properties are data.

- 开始思考名字的问题了...
- `sd-each` to `s-repeat`
- `additional data properties`是处理data属性的么？
- lazy = lazyload? async load?
- `v-model`前身
- escape {{{}}} 这些不会被解析
- 最后一点特性很给力，也是现在的`vue`有的特性：$开头的是库属性/方法；_开头是用户属性/方法；这两种开头的都不会被`observe`

---

#### move all API methods to Seed [65faa95](https://github.com/vuejs/vue/commit/65faa95549b100e71a69ad40c4919eb836dee2ca)

这是... Find and replace吗...

--

#### rename `prop` option to `scope` [99b25b2](https://github.com/vuejs/vue/commit/99b25b299803c3723452c0668e8f5da38538e799)

data => scope

剧透的赶脚。。反正会改回来的。。

---

#### implement sd-model [1e90903](https://github.com/vuejs/vue/commit/1e90903da8294dda1750e72ae90dd311115601d5)

#### sd-model [b8781c5](https://github.com/vuejs/vue/commit/b8781c54eba8f68e830471e0a80747afc31e14c2)




`sd-model`统一代替了`sd-value`和`sd-checked`

- input:checkbox
- input:radio
- input:text, select (及其他)

这里对于`select`也这么玩？IE应该是不支持直接设select的value的

```js
update: function () {
    this.el.checked = value == this.el.value
}

```

---

#### add sd-pre [b121a00](https://github.com/vuejs/vue/commit/b121a00a954dd54e52050f3de5bacc81472c4377)

`sd-pre`这个directive的innerHTML也是会skip complilation的

---

#### IE9 compatibility [7b17f80](https://github.com/vuejs/vue/commit/7b17f80eeec308e5507744d9096b34c456b054e9)

> 这里对于`select`也这么玩？IE应该是不支持直接设select的value的

果然，注释笑喷

```js
else if (el.tagName === 'SELECT') {
    // you cannot set <select>'s value in stupid IE9
    ...
    o.selectedIndex = index
} 
```

---

#### remove option to change interpolate tags [b2c4882](https://github.com/vuejs/vue/commit/b2c4882fdaafd5d7e6ce095cda26c7f83bf352b6)

强势要求语法是{{}}，这种语法已经被广泛认同不需要再用户自定义，

换言之，这里不需要过度设计

---


#### remove context binding related stuff [af342b7](https://github.com/vuejs/vue/commit/af342b75c0e5a20c01f7a868a34f16130e174e1f)

是要把 get(ctx) {//do something to ctx} 这种东西干掉的节奏

---

#### 0.4.0 [b5bc232](https://github.com/vuejs/vue/commit/b5bc232ba25b6b86ec6c5cd709a50517563fd9ca)

总结一下0.3.0 => 0.4.0

- 各种测试覆盖
- template API改进, 以及其他代码配合
- 修复observer重复emit问题
- 用户自定义directive和filter
- each的性能优化,批量dom更新后再插入
- each(repeat)加入$index
- Seed.ViewModel.extend() => Seed.extend()
- sd-model 


## 0.4.0 （branch 0.10） done 

---
#### should ignore keys starting with $ or _ when observing an external ob… [3700d2c](https://github.com/vuejs/vue/commit/3700d2c1f02618ba30bde365a9dc6372f6339707)

之前没有修改这里，补上

---

#### simple no-data directive [094a1fe](https://github.com/vuejs/vue/commit/094a1feca117f224bcfe95c0bebfaccef25b7c38)

对于`no-data directive`，即是expression为空的directive
比如一个用户自定义的directive, 如`v-btnloading`代表一个在点击时会有loading效果的button

```
<button v-btnloading></button>
```
只要在bind或update时做相应的调用即可

---

#### sd-id [5fa908d](https://github.com/vuejs/vue/commit/5fa908d1a292ed5b248cd9047fee7eb77fc15713)

对于viewmodel定义一个id （又是剧透感，会被干掉的...）

---

#### fix sd-repeat + sd-viewmodel [fb6b4ea](https://github.com/vuejs/vue/commit/fb6b4eaca505f53a7421745951e2102d04a55237)

`examples/repeated-vms.html` 里的这种用法感觉有点奇怪...

---

#### fix dependency tracking for nested values [db22a2a](https://github.com/vuejs/vue/commit/db22a2a0239d3d1e9eb39f1453f3770d04384df8)

`complier.js` 会对复杂的binding做处理，调用`exp-parser.js`，返回getter和vars,利用vars进行createBinding

```js
return {
            getter: new Function(args),
            vars: Object.keys(hash)
        }
```

```js
 while (i--) {
    v = result.vars[i]
    if (!bindings[v]) {
        compiler.rootCompiler.createBinding(v)
    }
}
```
但作者可能是在上个commit里面发现了vars返回的问题 

如

`item.title + msg` => vars=['item', 'msg']

而实际应该需要： path=['item.title', 'msg']

于是实现就是把vars再搞一遍匹配 

```js
function getPaths (code, vars) {
    // code: 'item.title + msg'
    var pathRE = new RegExp("\\b(" + vars.join('|') + ")[$\\w\\.]*\\b", 'g')
    // 上面例子就是/\b(item|msg)[$\w\.]*\b/g
    return code.match(pathRE)
    // ['item.title','msg']
}

```
---

#### restructure for functional tests, use CasperJS [412873f](https://github.com/vuejs/vue/commit/412873f5d89fe865e7096df7f58a763c508c4b15)

函数测试使用了[CasperJS](https://github.com/casperjs/casperjs)

这个并没有听说过= =后续有时间可以了解下 

---

#### use Object.create(null) for better hash [91d8528](https://github.com/vuejs/vue/commit/91d852863cbf95d1df11a25aa4a8081f18888dc4)


注释说得很清楚了： `Object.create(null)`的结果是`prototype-less`

```
/**
 *  Create a prototype-less object
 *  which is a better hash/map
 */
function makeHash () {
    return Object.create(null)
}
```

这个问题我之前有看过...

作为一个哈希or字典来说，显然是Object.create的结果好；因为它不inherit anything,
更加纯净

举例：
```
var a = {}
var b = Object.create(null)

```
这样，如果你使用`a.toString` => function toString() { [native code]}

看起来，toString这个key像已经被原型方法**占用**了, a作为哈希或字典就有点**不纯洁**

定义var a = {}, 实际相当于：

```
var a = Object.create(Object.prototype)
```

---

#### change model event to input, and try to fix it for IE9 (not tested) [dd68ea2](https://github.com/vuejs/vue/commit/dd68ea2cc132ef8767e76a24b0d524f797bb6043)

对IE9的鄙视之情跃然纸上

```
// fix shit for IE9
// since it doesn't fire input on backspace / del / cut
if (isIE) {
    el.addEventListener('cut', self.set)
    el.addEventListener('keydown', function (e) {
        if (e.keyCode === 46 || e.keyCode === 8) {
            self.set()
        }
    })
}
```
---

#### add teardown option; tests for $destroy() [5ffa301](https://github.com/vuejs/vue/commit/5ffa30146773a62afdc2515029bfb208f2a5da26)

`teardown`这个option，如果传入了，就在`vm.$destroy`的时候调用 

---

#### move $index to VM instead of item, add $parent & $collection [db284ff](https://github.com/vuejs/vue/commit/db284ffa94bac19dff4111a255a7fd2664541579)

设置完`parent vm`的时候把`child vm`就搞到`parent vm`上去，而不是搞到整个`vm`上

---

#### test implementation [8ba58dd](https://github.com/vuejs/vue/commit/8ba58dd795184e89461c47fe282eaac3e6159e30)

#### transition working with sd-if + sd-repeat (TODO: $destroy) [36da0a7](https://github.com/vuejs/vue/commit/36da0a783c54c9b0f3fa730993c3b917b86ca5de)

内置directives中的show, visible,if等dom操作统一到了`transition.js`上

changeState: enter/leave的时候调用的函数

这个API设计看起来有些别扭。。

```js
/**
  *  stage:
  *  1 = enter
  *  2 = leave
  */
module.exports = function (el, stage, changeState, init) {
    ...
    if (stage > 0) { // enter
        ...
    } else { // leave
        cl.add(className)
        el.addEventListener(endEvent, onEnd)
    }
}
```
这里endEvent='transitionEnd'，but这个事件`stupid IE9`是不支持的哇 =  =

目测这种设计也会干掉...

---

#### meta [23c98cb](https://github.com/vuejs/vue/commit/23c98cb505489d237a65dcaeaecf0681e7a2af5a)


> - IE9+ (IE9 needs [classList polyfill](https://github.com/remy/polyfills/blob/master/classList.js) and doesn't support transitions)

看来是考虑了这一点了。。

---

#### sniff transitionend event [cd80fe4](https://github.com/vuejs/vue/commit/cd80fe4bbb49c1032f6d37b63d439c8ddcd6b4e6)

兼容性处理

```js
function sniffTransitionEndEvent () {
    var el = document.createElement('div'),
        defaultEvent = 'transitionend',
        events = {
            'transition'       : defaultEvent,
            'MozTransition'    : defaultEvent,
            'WebkitTransition' : 'webkitTransitionEnd'
        }
    for (var name in events) {
        if (el.style[name] !== undefined) {
            return events[name]
        }
    }
}
```
---

#### change sd-viewmodel to sd-component, allow direct use of Objects in S… [5ad8ede](https://github.com/vuejs/vue/commit/5ad8edeb085d46e954da0ecb42b9cb51bf1fbe12)



```
<div class="vm" sd-component="vm-test">{{vmMsg}}</div>

```


```js
var T = Seed.extend({
    components: {
        'vm-test': Seed.extend({
            scope: {
                vmMsg: 'component works'
            }
        })
    },
    ...
})

```

---

#### sd-on can now execute expressions [9dc45ea](https://github.com/vuejs/vue/commit/9dc45ea9f62820d88dcf0baf0e8c59e26ebd039b)

```
sd-on="click: this.a = 'b'"
```

---

#### travis [f584c84](https://github.com/vuejs/vue/commit/f584c844b767c9ee23e9c0292a2532dce9b55adf)

使用了[travis ci](#### travis [f584c84](https://github.com/vuejs/vue/commit/f584c844b767c9ee23e9c0292a2532dce9b55adf))

---

#### 0.5.0 [8ade1f4](https://github.com/vuejs/vue/commit/8ade1f4c25238d8e1b99b8851181ad8af242b384)


## 0.5.0 （branch 0.10） done 

0.4.0 => 0.5.0

- 主要加入的功能是transition
- 引入Casper.js和travis ci做持续集成

---

#### split multiple expressions by unquoted commas [88ebff6](https://github.com/vuejs/vue/commit/88ebff66558c6be890a1f694fdd8b09a20d7eaa7)

不带引号的逗号 

'ffsef + "fse,fsef"' 带引号的

'fsef,fsf:fsefsef' 不带引号的，需要split 

这种逗号的场景举例：

```js
sd-class="
        completed : todo.completed,
        editing   : todo == editedTodo
    "
```

---

#### fix sd-model selection position issue [3eba564](https://github.com/vuejs/vue/commit/3eba564120c7ee161bd855d4447d3f4565c68d71)

```js
 // if this directive has filters
// we need to let the vm.$set trigger
// update() so filters are applied.
// therefore we have to record cursor position
// so that after vm.$set changes the input
// value we can put the cursor back at where it is
```

如果有filter,那是需要执行filter方法的即update()

鼠标位置这个东西也要修。。这个修的意义是？暂时没明白

如果没有filter, lock = true时，`model.js`中的update()就不会执行了

```js
function () {
    // no filters, don't let it trigger update()
    self.lock = true
    self.vm.$set(self.key, el[attr])
    self.lock = false
}
```

---

#### only emit get events during deps parsing - improves perf [23bd49f](https://github.com/vuejs/vue/commit/23bd49fa0814b3a07317f1f8eb53c35ad25fe59f)

只在做`computed properties`的依赖分析时用到`emit`，优化性能

`dep-parser.js`

```js
parse: function (bindings) {
        ...
        observer.active = true
        bindings.forEach(catchDeps)
        observer.active = false
        ...
    }
```
在`defineProperty`时就会去看这个标志位active，不需要频繁emit

---

#### New ExpParser implementation [bc0fd37](https://github.com/vuejs/vue/commit/bc0fd377d57ba67f27e04f6fee57d9b1b66fa938)

`exp-parser.js`中新增getRel()

例如对于`sd-checked="a.b"`

这个函数的作用是找到一个vm property(a)的真正owner vm 

同时，如果没有当前的path(a.b)则通过compiler.createBinding(path) 创建一个

---

#### fix input event handler for Chinese input methods [4e6ad72](https://github.com/vuejs/vue/commit/4e6ad727094a1c0b3e30d04db0d8528fa82be0ec)

这个真是闻所未闻，hack原理又是啥？。。。没弄明白

先抄下来吧

```js
try {
    cursorPos = el.selectionStart
} catch (e) {}
// `input` event has weird updating issue with
// International (e.g. Chinese) input methods,
// have to use a Timeout to hack around it...
setTimeout(function () {
    self.vm.$set(self.key, el[attr])
    if (cursorPos !== undefined) {
        el.setSelectionRange(cursorPos, cursorPos)
    }
}, 0)
```

---

#### add `replace` option & tests [0419b05](https://github.com/vuejs/vue/commit/0419b05535baa7e1ac4e1d569f2d50973073916a)

如果传入的option里设置`replace=true`，原来在dom上的声明式标签会被完全替换掉

---

#### 一堆fixes & tests...

---

#### 0.6.0 - rename to VueJS [218557c](https://github.com/vuejs/vue/commit/218557cdec830a629252f4a9e2643973dc1f1d2d)


终于取了个`never used before`的名字，感觉是0.5.0~0.6.0最大的改动 = =

## 0.6.0 （branch 0.10） done 

---

#### lifecycle hooks [e04553a](https://github.com/vuejs/vue/commit/e04553a0a433322cbcf7738b1d9716d340d426de)

绝赞~！

生命周期钩子出现~ 

类似：

```
if (ready) {
    ready.call(vm, options)
}
```

vm和options作为arguments可以进行操作。

- beforeCompile / created
- afterCompile / ready
- beforeDestroy
- afterDestroy


---

#### component refactor [628c42c](https://github.com/vuejs/vue/commit/628c42cc9f33172538d672428a67f5ef041a633a)

`src/directives/component.js` 专门用于处理组件化 

原来的自定义`elements`被干掉

`test/functional/fixtures/encapsulation.html` 来看，现在的组件化方式还不太友好。

---

#### DOM convenience methods [d132fdc](https://github.com/vuejs/vue/commit/d132fdc500572a56380bed917341a986e60befbc)

- $appendTo
- $remove
- $before
- $after
- query （querySelector）

---

#### enteredView/leftView hooks & test [9a132a9](https://github.com/vuejs/vue/commit/9a132a96375f910e160e3a0e8d39a931553cb38b)

增加一对transition的钩子：

- enterView
- leftView

---

#### v-model for content editable [9ec0259](https://github.com/vuejs/vue/commit/9ec0259df6b129ae2271ca8cf179fe6ecb07e572)

v-model现在可以附加到`textarea`上 

--- 

#### dom method callbacks should be async. [070d5c0](https://github.com/vuejs/vue/commit/070d5c0e583a963379dd90f9d3169b590ffaf98d)

出现了`utils.nextTick()`利用`requestAnimationFrame` 加载dom method的callback

---

#### observer rewrite [216e398](https://github.com/vuejs/vue/commit/216e398b22b835eb4b42b630f0943b86e98c08dd)

defineProperty的set中先做了unoserve oldVal,再observe newVal

```
set: function (newVal) {
    var oldVal = values[key]
    unobserve(oldVal, key, observer)
    values[key] = newVal
    ensurePaths('', newVal, oldVal)
    observer.emit('set', key, newVal)
    observe(newVal, key, observer)
}
```
---

#### add benchmark for todomvc example [dc17a4e](https://github.com/vuejs/vue/commit/dc17a4eda865ca5e4c4830d4cdfe6aa6a4400322)

加入了一个todomvc的简单benchmark

---

#### update benchmarks, remove attach/detach in v-repeat as its actually expensive in common use cases [f29be01](https://github.com/vuejs/vue/commit/f29be011db0152d2b21bd509bc95ff7640c5a4df)

> remove attach/detach in v-repeat as its actually expensive in common use cases

attach和detach不是为了批量更新dom而生的吗？为啥又干掉了？

实际使用场景里面很少有大批量更新dom的情况？。。。

---

#### compiler rewrite - observe scope directly and proxy access through vm [14d8ce2](https://github.com/vuejs/vue/commit/14d8ce24a3e4063dce241ea8daae15cce14e4c85)

`complier.js`做了比较大的改动 

scope现在可以依附在complier身上了 

先拷贝scope到vm里

在hook后user可能改变了vm, 需要再搞一遍vm到scope 

然后直接Observer.observe(scope, '', complier.observer)

即进行path,def,emit等事情 

```js

 // init scope
var scope = compiler.scope = options.scope || {}
utils.extend(vm, scope, true)

...

// beforeCompile hook
compiler.execHook('beforeCompile', 'created')
// the user might have set some props on the vm 
// so copy it back to the scope...
utils.extend(scope, vm)
// observe the scope
Observer.observe(scope, '', compiler.observer)
```

---

#### simplify v-repeat syntax [cd90e64](https://github.com/vuejs/vue/commit/cd90e647a7497b4eb3df0e85fcc497064d2005ce)

#### simplify v-component usage [c75aa42](https://github.com/vuejs/vue/commit/c75aa42d77c4ae6c9f5adaf18b88062bc0f29cd0)

改了之后更难用的即视感。。。

---

#### API change: split `scope` into `data` and `methods` [7c1d196](https://github.com/vuejs/vue/commit/7c1d196355c3d26e34b29b15d37347639f5f226a)

> 开发者的大事，大快所有人心的大好事

---

#### "Release v0.7.0" [56fbe04](https://github.com/vuejs/vue/commit/56fbe047ff07922831d48853fb97c7318a32772d)

## 0.7.0 （branch 0.10） done 

---

#### async batch update - first pass [5c73a37](https://github.com/vuejs/vue/commit/5c73a378ac1f67d05a3bf83816b4dfcde247bd0c)

异步的批量update, `batch.js`实现得很简洁 

binding.js在 update or refresh时均会使用batch.queue()

另外，viewmodel.js中transition的异步callback部分 

把requestAnimationFrame 换成了 setTimeout(func, 0)

猜测可能是更快的异步执行？

 因为requestAnimationFrame每秒只能触发60以下次（60FPS）

setTimeout(func, 0)可以触发200+次 （最小间隔约是4ms~10ms)。

---

#### nextTick phantomjs fix, unit tests for batcher, config() api addition [c7b2d9c](https://github.com/vuejs/vue/commit/c7b2d9ca347ad8a3171fdb82cc1a2f343c92a07f)

搞回了`nextTick`, 并在`batcher.js`里面换回了nextTick

仍不知道作者使用setTimeout和requestAnimationFrame之间的标准。。

---

#### avoid duplicate Observer.convert() [331bcc6](https://github.com/vuejs/vue/commit/331bcc673f143c9034b2fa78fc3a66e09a59cd48)

```
// if the data object is already observed, but the key
// is not observed, we need to add it to the observed keys.
if (ob && !(key in ob.values)) {
    Observer.convert(data, key)
}
```

比如在data和methods里面都没有定义的key出现了，那么要使用Observer.convert

如： 

```
{
    data:{},
    methods: {
        a: function () {
            // this.b没有被定义过
            var value = this.b && this.b;
            if (!value) value = this.b = 'c'
            return value
        }
    }
}
```

如果不加入这个条件，因为data/methods中的key都已经observed，会重复observe

---

#### test using gulp plugins instead [c7f8a68](https://github.com/vuejs/vue/commit/c7f8a68d4afc367915820de402b18095f696b8ea)

部分test用上了gulp, 时髦啊...

---

#### Async fixes [c71a908](https://github.com/vuejs/vue/commit/c71a9085a74befcce429840ad2c36a26d2f15d34)

vm.$watch 现在支持异步
v-model不支持异步是个原先的bug. 这里修复了

---

#### perfectly resolve Chinese input methods issue with composition events [1e58721](https://github.com/vuejs/vue/commit/1e587210d7501434f4120f21923ff4b7118e2356)

利用`compositionstart`和`compositionEnd`事件解决中文输入问题 

不知道有多少人跟我反应一样“卧槽居然还有这事件”

学习了一下[compositionEvents](https://developer.mozilla.org/en-US/docs/Web/API/CompositionEvent)

mark~

---

#### meta [74e2f38](https://github.com/vuejs/vue/commit/74e2f389309cac164a000916296d57c7b28903ab)

作者的开发思想

> Technically, it provides the **ViewModel** layer of the MVVM pattern, which connects the **View** (the actual HTML that the user sees) and the **Model** (JSON-compliant plain JavaScript objects) via two way data-bindings.
> 
> Philosophically, the goal is to allow the developer to embrace an extremely minimal mental model when dealing with interfaces - there's only one type of object you need to worry about: the ViewModel. It is where all the view logic happens, and manipulating the ViewModel will automatically keep the View and the Model in sync. Actuall DOM manipulations and output formatting are abstracted away into **Directives** and **Filters**.

---

#### vm.$emit() is now self only, propagating events now triggered by $dispatch() [354f559](https://github.com/vuejs/vue/commit/354f559215bb3f01707bec23877cff47929da719)

新增`$emit`用于触发自己的事件，`$dispatch`用来触发冒泡事件，和现在一致了

---

#### add support for interpolation in html attributes [a2e287f](https://github.com/vuejs/vue/commit/a2e287f0b98a6ec49d330d23b55d4daf20d20535)

允许在正常的html属性里面带上{{}}

> // non directive attribute, check interpolation tags

like this: 

```
<img src="{{url}}"></img>
```

---

#### make exp-parser deal with cases where strings contain variable names [212990a](https://github.com/vuejs/vue/commit/212990a3bb10e4a6d5554dbd20300b7a336a46be)

> exp: "'\"test\"' + test + \"'hi'\" + hi"

我觉得这算是一种edge case了, 真这么用的人...

---

#### Component Change [04249e3](https://github.com/vuejs/vue/commit/04249e320ab5aa31fe64c4657d9a319b25c0b311)

- `v-component` now takes only a string value (the component id)
- `Vue.component()` now also registers the component id as a custom element
- add new directive: `v-with`, which can be used in combination with `v-component` or standalone

---

#### Fix inline partial parentNode problemDirectives inside inline partials would get the fragment as the parentNodeif they are compiled before the partial is appended to the correct parentNode.Therefore keep the reference of the fragment's childNodes, append it,then compile afterwards fixes the problem. [5f21eed](https://github.com/vuejs/vue/commit/5f21eed2c9ed135d55e7d3d1eed696522bebe68e)

对于partial的处理，在append fragment之后再进行compile

否则其children的parentNode会不对 

---

#### bindings that are not found are now created on current compiler instead of root [1fb885b](https://github.com/vuejs/vue/commit/1fb885b9fa312bb5bcf5a679310683bd7793212a)

当一个baseKey(如a.b的a)不在当前vm的时候，之前是去rootComplier.createBinding,

现在是递归地找complier.parentComplier, 就近地createBinding

---

#### clean up get binding owner compiler logic [9a45f63](https://github.com/vuejs/vue/commit/9a45f63f343da1f0e41b0d9e2896713e6378089d)

`compiler.bindings = makeHash()` 这样初始化 

后面在function check(key) 中就不用再去检查hasOwnProperty

---

#### should lock during composition in all cases [61e4dd1](https://github.com/vuejs/vue/commit/61e4dd13baebe748b40a186da89b63096b911c09)

以前是仅当Expression中带有filter时才进行`composition lock`

现在是所有的情况均进行lock

---

#### update() is now optional when creating a directive [d3ebd42](https://github.com/vuejs/vue/commit/d3ebd42dd86a4abb4e6bbe7136833c1943726ec1)

当进行一个自定义directive时，有可能只是初始化一次(once)之后就不再变化了 

所以这个update是optional的

---

#### simplify binding mechanism, remove refresh() [f139f92](https://github.com/vuejs/vue/commit/f139f9271a0aefe25d5f7d4f12844143ec35ea7b)

原来对于`computed properties`，在数据更新时用refresh()方法； 

现在用一个eval()把所有的properties都合并到一块统一使用_update()

```js
/**
 *  Return the valuated value regardless
 *  of whether it is computed or not
 */
BindingProto.eval = function () {
    return this.isComputed && !this.isFn
        ? this.value.$get() // is computed
        : this.value // not computed
}

```

---

#### do not modify original computed property getter/setters [62fd49a](https://github.com/vuejs/vue/commit/62fd49abd231b1bbce29e099abcd2367d97ef077)

修改发生在`CompilerProto.define`即说明是个`root-level binding` (即非expression, 不是a.b,或带$parent/$root) 的

对于这种binding, ` Object.defineProperty`的时候不设置`enumerable`即默认为false

compiler.data即是原始用户传入的data部分

```js

Object.defineProperty(vm, key, {
    get: binding.isComputed
        ? function () {
            // 原先是compiler.data[key].$get
            return binding.value.$get()
        }
        : function () {
            return compiler.data[key]
        },
    set: binding.isComputed
        ? function (val) {
            // 原先是compiler.data[key].$set
            if (binding.value.$set) {
                binding.value.$set(val)
            }
        }
        : function (val) {
            compiler.data[key] = val
        }
})

```



---

#### separate computed properties into `computed` option [7a6169c](https://github.com/vuejs/vue/commit/7a6169caf9aed817af2779b1f9beb3bc21550ad4)

很牛逼的举动，

- 即方便了用户（后续还可以不用去死板地写$get function), 也可以和data/method区分开来，方便代码组织

- 也方便了自己（computed中的东西必须是computed property）

---

#### refactor Compiler.createBinding [50a7881](https://github.com/vuejs/vue/commit/50a7881b2f6efd550630543770f2752a2cfd83e6)

vm中加入了computed之后， createBinding做一次重构 

看下面这个代码多么的清晰！


```js
if (binding.root) {
    // this is a root level binding. we need to define getter/setters for it.
    if (computed && computed[key]) {
        // computed property
        compiler.defineComputed(key, binding, computed[key])
    } else {
        // normal property
        compiler.defineProp(key, binding)
    }
} 

```

CompilerProto.define => CompilerProto.defineProp + CompilerProto.defineExp 

分而治之，代码也更加清晰易读，赞~ 

---


#### Release-v0.8.0 [882c16c](https://github.com/vuejs/vue/commit/882c16c76ec0790ccf60856385ea493c9369cd63)

## 0.8.0 （branch 0.10） done 


---

#### defer child components compilation [3cb7027](https://github.com/vuejs/vue/commit/3cb7027725a548acb0776a6bf5d983f4058bf73b)

把child components延迟compile,这样在child components进行compile时，其parent已经收集了所有的binding

---

#### Fix #65A 
computed property can depend on properties on repeated items.When new items are added to the Array, the dependency collection has already been done so their properties are not collected by the computed property on the parent. We need to re-parse the dependencies in such situation, namely push, unshift, splice and when the entireArray is swapped. Also included casper test case by @daines. [a7dfb3f](https://github.com/vuejs/vue/commit/a7dfb3f4c89b84cc6dcfbb7ca17c1e4caa06d5f6)

针对这样的case:

```
<body>
    <form id="form">
        <p v-repeat="items">
            <input type="text" name="text{{$index}}" v-model="text">
        </p>
        <button v-on="click: add" id="add">Add</button>
        <p id="texts">{{texts}}</p>
    </form>
    <script>
        var app = new Vue({
            el: '#form',
            data: {
                items: [
                    { text: "a" },
                    { text: "b" }
                ]
            },
            methods: {
                add: function(e) {
                    this.items.push({ text: "c" })
                    e.preventDefault()
                }
            },
            computed: {
                texts: function () {
                    return this.items.map(function(item) {
                        return item.text
                    }).join(",")
                }
            }
        })
    </script>
</body>
```

当点击add button时，input会多加一个:

input:a
input:b
input:c

但是在input:c中输入文字的时候，{{texts}}没有改变 

原因作者说得很清楚了。在这个例子里面texts是一个computed property 

当array push的时候，依赖收集已经完成,只对input:a和input:b有双向绑定

那么在push完成之后，需要对input:c也有双向绑定

对于 `push/unshift/splice` 这些改变数组长度的行为 ：

mutationListener => changed()里面再进行一次parseDeps()

```js
elf.mutationListener = function (path, arr, mutation) {
    ...
    if (method === 'push' || method === 'unshift' || method === 'splice') {
        self.changed()
    }
}

...

/**
    *  Notify parent compiler that new items
    *  have been added to the collection, it needs
    *  to re-calculate computed property dependencies.
    *  Batched to ensure it's called only once every event loop.
    */
changed: function () {
    var self = this
    if (self.queued) return
    self.queued = true
    setTimeout(function () {
        self.compiler.parseDeps()
        self.queued = false
    }, 0)
},
```

---

#### text-parser deal with triple mustache [1c6dacf](https://github.com/vuejs/vue/commit/1c6dacf5b0bc56b27f593a67f96f855d4a57ff6d)

#### {{{ }}} tags for unescaped html [58dd07e](https://github.com/vuejs/vue/commit/58dd07e5756eb9bc7c5d004985dd1122659db0c8)

这两个commit解决`{{{xxx}}}`的问题，如果expression match则里面的xxx当做html来解析

---

#### rewrite lifecycle hook system [d75ee62](https://github.com/vuejs/vue/commit/d75ee6234e8729694b61f4b63a73f40a8aa27711)

每个hook就叫一个名字比较好...

---

#### make sure a vm and its data are properly removed from Array when $destroy is called directly [efcaad1](https://github.com/vuejs/vue/commit/efcaad1b8bc48ec482b9dc45077c3ae03a5fef40)

当$destroy的时候,vm和data都要从collection里面清除掉 

---

#### reintroduce v-style [120af4b](https://github.com/vuejs/vue/commit/120af4b77436b9100d93165274c793662684ebc1)

style从新搞回来了

---
#### update todomvc example to be same as lended version [cb1d69c](https://github.com/vuejs/vue/commit/cb1d69c66c10ba224eeeb3af19689d450c0fce2b)

尤大带逛github系列：

后面的`vue-router`是否也参考了这个[director](https://github.com/flatiron/director)呢。。。

这东西最后一个commit已经是2 years ago了

先mark，待研究

---

#### add isLiteral option for custom directive [6673828](https://github.com/vuejs/vue/commit/66738285422aea7ce6fdd5ca500b5fc17bba876b)

expression仅是字面量的场景应该很少吧

---

#### Fix IE directive not compiled issueIn the TodoMVC example, when `v-show` is compiled first, it adds aninline style attribute to the node. Since IE seems to handle the orderof attributes in `node.attributes` differently from other browsers,this causes some directives to be skipped and never compiled. Simplycopy the attribtues into an Array before compiling solves the issue. [0f448b8](https://github.com/vuejs/vue/commit/0f448b8678b2d64a87632783761d01eba668cb27)

又是IE9的奇怪bug 

---

#### common xss protection [9c2bb94](https://github.com/vuejs/vue/commit/9c2bb9465e4fba4f36d7be3986ae4bf1b484f4da)

防了下constructor的override攻击

例如这个：

```
toString.constructor.prototype.toString=toString.constructor.prototype.call;

["a","alert(1)"].sort(toString.constructor);
```

---

#### select[multiple] support [e16f910](https://github.com/vuejs/vue/commit/e16f910948e50968cba914b704fc844a8dc92a8e)

支持 select multiple属性，但是。。。这个估计也很少人会用吧 

multiselect基本所有人都是用组件的...原生的太坑了 

---

#### plugin interface [a891df2](https://github.com/vuejs/vue/commit/a891df220f003c729b57bf4d84b4c3b50bc7b462)

又一个被广泛应用的接口出现了 = = 

use和install => 直接操作viewModel 

---

#### better xss mitigation [0bee5a0](https://github.com/vuejs/vue/commit/0bee5a08e5c3caddfbba4b536e62dd5c8954bc73)

在`constructor`基础上加入了防止`unicode`形式的攻击 


---
#### revert repeated item $destroy behavior [ce4342b](https://github.com/vuejs/vue/commit/ce4342bbfce16ec68ea1e5673fa3a8d8ddc814e0)

没想明白为什么这部分又搞回去了。。。=  =

```
// in case `$destroy` is called directly on a repeated vm
// make sure the vm's data is properly removed

item.$compiler.observer.on('hook:afterDestroy', function () {
    col.remove(data)
})
```

---

#### remove isLiteral option [f6ee24a](https://github.com/vuejs/vue/commit/f6ee24a5cfb05ffd80e7b66bf3d0fdaa1ef0431c)

> expression仅是字面量的场景应该很少吧

果然干掉了 (＞◡❛)

---

#### add v-cloak [c599bed](https://github.com/vuejs/vue/commit/c599bed93d4fab264075aacef0496418ab7dcd39)

`v-cloak` 作用主要是和`display:none`结合使用，编译完成后才展示

要不一进来会是{{}}这种标签展示在界面上的话不太好看的嘛

---
#### Make Observer emit `set` for all parent objects too [f2e32ab](https://github.com/vuejs/vue/commit/f2e32ab8ceec092e5f8acbc5c71358a1c1ec51e8)

#### add tests for object outputing + avoid duplicate events [77ffd53](https://github.com/vuejs/vue/commit/77ffd53b6b0d1b0d964ab7192a1d7e59acaaddc0)

---

#### v-repeat optimization[9482e51](https://github.com/vuejs/vue/commit/9482e51250f9fe0a0e1a504143340d7920f7efc9)
> - When the array is swapped, will reuse existing VM for any  element present in the old array. This greatly improves  performance when the repeated VMs have a complicated  nested structure themselves.
> - Fixed issue when array is swapped new length is not emitted
> - Fixed v-if not removing its reference comment node when unbound
> - Fixed transition when element is invisible the transitionend  callback is never fired resulting in element stuck in DOM 

这个厉害了：

在array改变的时候执行update时会进行buildItem

之前是全量地buildItem, 那么就需要全量地创建childvm

现在在directive update时先存来一份: `this.oldVMs = this.vm`

这样在buildItem时：

```js
 buildItem: function (data, index) {
        ...
        // append node into DOM first
        // so v-if can get access to parentNode
        if (data) {

            if (this.old) {
                i = this.old.indexOf(data)
            }
            
            if (i > -1) { // existing, reuse the old VM

                item = this.oldVMs[i]
                // mark, so it won't be destroyed
                item.$reused = true
                el = item.$el
                // don't forget to update index
                data.$index = index
                // existing VM's el can possibly be detached by v-if.
                // in that case don't insert.
                noInsert = !el.parentNode

            }
            ... 
        }
}
```

上面标记了一个$reused, 说明可以被复用；

update => buildItem => 处理oldVM

```js
        // destroy unused old VMs
        if (oldVMs) {
            var i = oldVMs.length, vm
            while (i--) {
                vm = oldVMs[i]
                if (vm.$reused) {
                    vm.$reused = false
                } else {
                    vm.$destroy()
                }
            }
        }
```

---

#### v-repeat object first pass, value -> $value [393a4e2](https://github.com/vuejs/vue/commit/393a4e2cfa72ff6d9d91611edae52c579f214ed3)

#### repeat object sync and tests [0e486a3](https://github.com/vuejs/vue/commit/0e486a30a26716fdc4d761a38e318de23e5fa8bb)

对于v-repeat object,利用for in搞成array （这里定义成一个$repeater property,值是 array)

在object更新时，$repeater array也进行更新 

---

#### sync back inline input value if initial data is undefined [31f3d65](https://github.com/vuejs/vue/commit/31f3d6598474fe7ae1d3f4bd9a87be66768b6b6d)

举例：

```
<input type="checkbox" v-model="a">
```

在init的时候a是undefined的话，把checked属性置为html的默认值 

---

#### allow components to use plugins too, resolve Browserify Vue.require issue [74c03f3](https://github.com/vuejs/vue/commit/74c03f3e126050064755639700f632f6aacca4e5)

ViewModel.use/require 挂到ExtendedVM上，任何vm(或者理解为components)都可以使用plugin

```
  // allow extended VM to use plugins
   ExtendedVM.use     = ViewModel.use
   ExtendedVM.require = ViewModel.require
```

---

#### Internalize emitter implementation [3693ca7](https://github.com/vuejs/vue/commit/3693ca7f5265d1fdb48d8656160991e1ac023177)

> This has a few benefits
>
> - no longer need to shiv the difference between Component's emitter  & Browserify's emitter (node version)
> - no longer need to hack around Browserify's static require parsing> -able to drop methods not used in Vue
> - able to add custom callback context control, as mentioned in #130 

迟早要自己实现emitter的！ 写个emitter对尤大来说简直易如反掌...

这样可以脱离这个Component的控制了，这东西(生态)后续也被证明是不成气候的

---

#### dataAttributes options (#125) [659593f](https://github.com/vuejs/vue/commit/659593f0e6fb0863b6029d7288788953ff4cc08a)

这种paramAttributes声明方式又是迟早被干掉的节奏（剧透脸）

```js
describe('paramAttributes', function () {
    it('should copy listed attributes into data and parse Numbers', function () {
        var Test = Vue.extend({
            template: '<div a="1" b="hello"></div>',
            replace: true,
            paramAttributes: ['a', 'b']
        })
        var v = new Test()
        assert.strictEqual(v.a, 1)
        assert.strictEqual(v.$data.a, 1)
        assert.strictEqual(v.b, 'hello')
        assert.strictEqual(v.$data.b, 'hello')
    })

})
```

---

#### v-on delegation refactor, functional tests pass [60e5154](https://github.com/vuejs/vue/commit/60e5154715b03fc2bc6e302d65d882654738fe29)

总体思路就是把事件代理到viewmodel的el上了

触发时递归去找，找到了执行handler

```js
CompilerProto.addListener = function (listener) {
    var event = listener.arg,
        delegator = this.delegators[event]
    if (!delegator) {
        // initialize a delegator
        delegator = this.delegators[event] = {
            targets: [],
            handler: function (e) {
                var i = delegator.targets.length,
                    target
                while (i--) {
                    target = delegator.targets[i]
                    // 这里应该是target.el.contains(e.target)
                    if (e.target === target.el && target.handler) {
                        target.handler(e)
                    }
                }
            }
        }
        this.el.addEventListener(event, delegator.handler)
    }
    delegator.targets.push(listener)
}
```

---

#### setTimeout(0) is faster than rAF [5100f3e](https://github.com/vuejs/vue/commit/5100f3ee0c93f18c2d9d8dde2273e16c4c6da5fe)

前面的一个commit笔记  [c7b2d9c](https://github.com/vuejs/vue/commit/c7b2d9ca347ad8a3171fdb82cc1a2f343c92a07f)


> 仍不知道作者使用setTimeout和requestAnimationFrame之间的标准。。

从这个commit来看，还是速度上的考虑？

---

#### make primitive arrays work with computed property [8d65a07](https://github.com/vuejs/vue/commit/8d65a078f3c1bdf82e655e3745508533b6c15df3)

`src/repeat.js`

change:$value时再set一次，触发computed property更新

```js
// for non-object values, listen for value change
// so we can sync it back to the original Array
if (nonObject) {
    item.$compiler.observer.on('change:$value', function (val) {
        self.lock = true
        self.collection.set(item.$index, val)
        self.lock = false
    })
}
```

---

#### allow nested path for computed properties that return objects [bedcb22](https://github.com/vuejs/vue/commit/bedcb22f9c08a29fd3c2399658775a6b0908a8fe)

在进行createBinding时：如果这个path是nested path(带.的) 

继续进行一次defineExp, 当做表达式来处理 

```js
else if (computed && computed[key.split('.')[0]]) {
            // nested path on computed property
            compiler.defineExp(key, binding)
        }
```

---

#### filterBy & orderBy first pass [336d06d](https://github.com/vuejs/vue/commit/336d06de1d0bddf7159d17db0cd218ca7a099358)

对数组加入filterBy和orderBy的filter


---

#### v-component & v-with refactor [665520f](https://github.com/vuejs/vue/commit/665520f59b7e6cb33bbfc3c6ae0d8d772941e550)

v-component也成了一个内置directive


---


#### refactor compile priority order [775948d](https://github.com/vuejs/vue/commit/775948d31f55a8638f59d5830f2136f294a71731)

compile的先后顺序: repeat => view => component

--- 


#### array diff WIP [a847fdd](https://github.com/vuejs/vue/commit/a847fddc2f7b1a66aa04e77886d6b9a4bd6ed012)

#### clean up array diff algorithm [ed0be36](https://github.com/vuejs/vue/commit/ed0be36de3866a4829d48277c40fc8c6f90febf6)

大动作~！

`src/repeat.js` 中的diff()方法：

在Array update时，去跟原Array进行对比，做到**最小力度**的dom操作

可以看做一种`v-dom`的思想吧

原来的`mutationHandlers`全部干掉了，因为都可以理解成是一种diff

update时：

```js
 this.vms = this.oldCollection
            ? this.diff(collection, isObject)
            : this.init(collection, isObject)

```
来完整地看一下diff方法的实现：

```js
/**
    *  Diff the new array with the old
    *  and determine the minimum amount of DOM manipulations.
    */
diff: function (newCollection, isObject) {

    var i, l, item, vm,
        oldIndex,
        targetNext,
        currentNext,
        ctn    = this.container,
        oldVMs = this.oldVMs,
        vms    = []
    // vms是diff后的结果Array,给个长度先
    vms.length = newCollection.length

    // first pass, collect new reused and new created
    // 第一步是先把可以复用vm的和新创建的搞进来
    for (i = 0, l = newCollection.length; i < l; i++) {
        item = newCollection[i]
        if (isObject) { // 对Object类型的处理
            item.$index = i
            // identifier存在，那么是可以复用的
            if (item[this.identifier]) {
                // this piece of data is being reused.
                // record its final position in reused vms
                item.$reused = true
            } else {
                // 这些就是新建的
                vms[i] = this.build(item, i, isObject)
            }
        } else { // 这就是Array了 
            // we can't attach an identifier to primitive values
            // so have to do an indexOf...
            oldIndex = indexOf(oldVMs, item)
            if (oldIndex > -1) { // 说明在oldVM里面存在，可以复用
                // record the position on the existing vm
                oldVMs[oldIndex].$reused = true
                oldVMs[oldIndex].$data.$index = i
            } else {
                // 新建
                vms[i] = this.build(item, i, isObject)
            }
        }
    }

    // second pass, collect old reused and destroy unused
    // 第二步，处理oldVM里面没有被reused的东西，干掉
    for (i = 0, l = oldVMs.length; i < l; i++) {
        vm = oldVMs[i]
        item = vm.$data
        if (item.$reused) {
            vm.$reused = true
            delete item.$reused
        }
        if (vm.$reused) {
            // update the index to latest
            // index要保持住
            vm.$index = item.$index
            // the item could have had a new key
            // 参见objectToArray, 这个$key可能已经改变了
            if (item.$key && item.$key !== vm.$key) {
                vm.$key = item.$key
            }
            vms[vm.$index] = vm
        } else {
            // this one can be destroyed.
            delete item[this.identifier]
            vm.$destroy()
        }
    }

    // final pass, move/insert DOM elements
    // 最后一步做dom操作
    i = vms.length
    while (i--) {
        vm = vms[i]
        item = vm.$data
        targetNext = vms[i + 1]
        if (vm.$reused) {
            // 搞位置，插入
            currentNext = vm.$el.nextSibling.vue_vm
            // 如果currentNext === targetNext说明没有必要做dom操作了
            if (currentNext !== targetNext) {
                if (!targetNext) {
                    ctn.insertBefore(vm.$el, this.ref)
                } else {
                    ctn.insertBefore(vm.$el, targetNext.$el)
                }
            }
            delete vm.$reused
            delete item.$index
            delete item.$key
        } else { // a new vm
            // 新创建的，insertBefore
            vm.$before(targetNext ? targetNext.$el : this.ref)
        }
    }

    return vms
},
```

---

#### enable $event in v-on expressions + enable e.preventDefault() [1b1d72e](https://github.com/vuejs/vue/commit/1b1d72ea5680e29eaa33c1648727dc19857bb8df)

在dom上触发事件的时候把事件句柄e作为$event传入

那么就可以this.$event了

```js
newHandler = function (e) {
    e.targetEl = el
    e.targetVM = targetVM
    context.$event = e
    handler.call(context, e)
    context.$event = null
}
```
---

#### remove event delegation [423b54d](https://github.com/vuejs/vue/commit/423b54dfd41571fe45e7c46d465ad75665dc1b82)

咋又干掉了？

---

#### also propagate array mutations in observer [e498191](https://github.com/vuejs/vue/commit/e49819118b1efed522bf81e664e03e15816d5d07)


这是把dom的冒泡转移到了array data emitter的emit上吗。。。感觉很牛逼的样纸 =  =

```js

emitter
    .on('set', function (key, val, propagate) {
        if (propagate) propagateChange(obj)
    })
    .on('mutate', function () {
        propagateChange(obj)
    })

...

/**
 *  Propagate an array element's change to its owner arrays
 */
function propagateChange (obj) {
    var owners = obj.__emitter__.owners,
        i = owners.length
    while (i--) {
        owners[i].__emitter__.emit('set', '', '', true)
    }
}

```

---

#### expression cacheexpressions with the same signature are now cached! this dramaticallyimproves performance on large v-repeat lists with multiple expressions. [15a6733](https://github.com/vuejs/vue/commit/15a673388e14c1c9e2ef196aaee28f9a5cc17d4d)

```js
compiler.expCache = compiler.expCache || makeHash()
...

if (!getter) {
        getter = this.expCache[exp] = ExpParser.parse(key, this, null, filters)
    }
```

ExpParser.parse()后的结果被缓存，避免不必要的重复parse，提高性能 

---

#### add tree view test case [3be1141](https://github.com/vuejs/vue/commit/3be1141081bec448a6a54c495f8391403791024d)

递归使用组件，cool

---

#### use more efficient Object check [c67685b](https://github.com/vuejs/vue/commit/c67685b713026ca0956e71adbd21ec5de3950fef)

基本类型判断 

这两种东东看来是有Benchmark上的区别？

也要mark一下。。。

```js
/**
    *  A less bullet-proof but more efficient type check
    *  than Object.prototype.toString
    */
isObject: function (obj) {
    return typeof obj === OBJECT && obj && !Array.isArray(obj)
},

/**
    *  A more accurate but less efficient type check
    */
isTrueObject: function (obj) {
    return toString.call(obj) === '[object Object]'
},
```

---

#### Release-v0.10.0 [cd53688](https://github.com/vuejs/vue/commit/cd53688d5364aef64aafc986c02e491f8cf90f82)


## 0.10.0 （branch 0.10） done 

---
