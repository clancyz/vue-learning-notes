# Vue learing notes

My note of learning Vue.js

## 0.1.0

### 16/9/2
- initial setup [83fac01](https://github.com/vuejs/vue/tree/83fac017f96f34c92c3578796a7ddb443d4e1f17)

    `Vue` 最初的名字叫 `element` 。
    
- rename [871ed91](https://github.com/vuejs/vue/tree/871ed9126639c9128c18bb2f19e6afd42c0c5ad9)

    改名叫 `seed` ，雏形出现。
    
    对`Mustache`语法 `{{}}` 做一个replace,替换掉原来的innerHTML
    
    ```html
    <p>{{msg}}</p>
    // parse后：
    <p><span data-element-binding="msg"></span></p>
    ```
    
    同时维护一个 `bindings` 变量，存储所有 `{{}}` 中的绑定变量，如 `msg`
    
    对 `msg` 维护一个`els`的属性，使用 `document.queryselectAll` 和前面parse后的html中的selector, 即 `data-element-binding="msg"` 拿到所有DOM，存入。然后删除 `data-element-binding` 这个属性。
    
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
 