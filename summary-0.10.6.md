# Vue.js 0.10.0分支源码学习总结

## 综述

Vue 在0.10.0分支共有737个commits, 历经时间 2013/7/29 - 2014/7/29（0.10.6release），正好是一年。 也许在作者心中，这也是个里程碑式的事件吧~ 

在0.10.6版本发布这个节点上，Vue已经是一个较完整的前端MVVM框架。而且开发体验也已经十分不错。主要包含的功能：

- Observer数据，利用`Emitter`和`getter/setter`实现mutation observer
- input/select等的用户输入源的双向绑定(v-model)
- `computed properties`自动计算属性，明显减少开发者的代码量
- 丰富的内置指令和基于`exp-parser`的expression自动计算，使view层(html)开发简单高效
- 自定义指令
- 可复用组件化设计
- 组件完善的生命周期设计
- 对开发者友好的viewmodel层开发范式
- 过渡系统(transition)便于视图切换

而且体积小（14kb Gzip），易与其他library结合，完善的测试覆盖，简单便捷的API设计，为Vue能有今天的辉煌打下了很好的基础。


## 目录结构

```
src
├── batcher.js // 用于批处理任务
├── binding.js // 绑定基础类，用于处理ViewModel上的各属性，生成Binding
├── compiler.js // DOM compiler
├── config.js // 配置，如prefix,debug,interpolate等属性
├── deps-parser.js // 依赖收集器 
├── directive.js // 指令基础类
├── directives // 内置指令
│   ├── html.js
│   ├── if.js
│   ├── index.js
│   ├── model.js
│   ├── on.js
│   ├── partial.js
│   ├── repeat.js
│   ├── style.js
│   ├── view.js
│   └── with.js
├── emitter.js // 事件发射器
├── exp-parser.js // 表达式parser，最终转义成可执行function
├── filters.js // 内置filters
├── fragment.js // fragment wrapper
├── main.js // ViewModel的公共方法
├── observer.js // 对象和数组 => Observerables
├── template-parser.js // template string => dom fragment
├── text-parser.js // 解析包含绑定的文本
├── transition.js // 过渡系统
├── utils.js // 工具函数
└── viewmodel.js // 视图类，管理数据的状态，事件等。
```

## 主流程

new Vue (options) => 
    
> util.processOptions()

- 解析template  => template parser
- 解析自定义filter => computed properties
- 解析components => Viewmodel.extend()
- 解析partials => template parser

> 初始化compiler

- vm, bindings, dirs, deferred, computed, children, emitter
- compiler.setupElement(options)

> 初始化vm

- $el, $options, $compiler

> 设置parentvm / root

> 为生命周期钩子和data设置observer

- observer为一个Emitter实例
- 在`get/set/mutate`时添加监听器，onGet/onSet => compiler.createBinding(key)
- 注册生命周期钩子

> 处理options.methods

- compiler.createBinding(key)

> 处理options.computed 

- compiler.createBinding(key)

> 初始化data

- copy paramAttributes 
- copy data 到 vm, vm.$data = data

> `beforeCompile` 钩子 compiler.exeHook('created')

> 处理用户在`beforeCompile`中（可能）改变的data

> 现在可以进行observeData了： compiler.observeData(data)

- 递归地观察`nested properties`
- 第一级的`$data` => new Binding(compiler, '$data') => update
- $data可替换，也需要def(get,set)

> 开始解析dom, 如果有options.template, 处理之

> 解析dom和指令 compiler.compile(el, true)

- this.compileElement(node, root)
    - 有属性的node 或者是tagname带`-`的
    - check directive 优先级
    - parseDirective
    - bindDirective
- this.compileTextNode(node)

> 开始处理有childVM的，这种被延迟处理（挂载在compiler.deferred）

- compiler.bindDirective()

> 依赖解析

- DepsParser.parse(this.computed)
    - getter trigger的时候，自动收集依赖

> init Done 

> 执行`ready`的生命周期钩子


## 主要技术难点

## 优化历程

## Tricks