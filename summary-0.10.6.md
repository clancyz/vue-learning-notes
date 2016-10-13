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
├── batcher.js
├── binding.js
├── compiler.js
├── config.js
├── deps-parser.js
├── directive.js
├── directives
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
├── emitter.js
├── exp-parser.js
├── filters.js
├── fragment.js
├── main.js
├── observer.js
├── template-parser.js
├── text-parser.js
├── transition.js
├── utils.js
└── viewmodel.js
```

## 主流程

## 主要技术难点

## 优化历程

## Tricks