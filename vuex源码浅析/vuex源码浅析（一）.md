# 			Vuex源码浅析

> 本文Vuex版本为3.0.1

Vuex是Vue的单一状态树,什么是单一状态树,简单来说就是Vuex把所有需要用到的状态放到了一个对象中,也就是单例模式;首先看一下Vuex的结构;

![](F:\blog\vuex源码浅析\源码结构.png)



## 源码分析

1. install

   > 安装 Vue.js 插件。如果插件是一个对象，必须提供 `install` 方法。如果插件是一个函数，它会被作为 install 方法。install 方法调用时，会将 Vue 作为参数传入。
   >
   > 该方法需要在调用 `new Vue()` 之前被调用。
   >
   > 当 install 方法被同一个插件多次调用，插件将只会被安装一次。

   从这段Vue的文档中我们可以推断出来Vuex一定提供了一个install方法,我们首先找一下install,

   ```javascript
   import { Store, install } from './store'
   import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'
   
   export default {
     Store,
     install,
     version: '__VERSION__',
     mapState,
     mapMutations,
     mapGetters,
     mapActions,
     createNamespacedHelpers
   }
   ```

   从中我们可以看到`install`方法在store文件中,转到store,

   ```js
   
   let Vue // bind on install
   export function install (_Vue) {
     if (Vue && _Vue === Vue) {
       if (process.env.NODE_ENV !== 'production') {
         console.error(
           '[vuex] already installed. Vue.use(Vuex) should be called only once.'
         )
       }
       return
     }
     Vue = _Vue
     applyMixin(Vue)
   }
   ```

    `_Vue`是Vue.use时传入的`Vue`构造函数,而`store`维护一个`Vue`变量,保证`vuex`只会被安装一次,而具体的安装操作则在`applyMixin`函数中,

   ```javascript
   export default function (Vue) {
     const version = Number(Vue.version.split('.')[0])
   
     if (version >= 2) {
       Vue.mixin({ beforeCreate: vuexInit })
     } else {
       // override init and inject vuex init procedure
       // for 1.x backwards compatibility.
       const _init = Vue.prototype._init
       Vue.prototype._init = function (options = {}) {
         options.init = options.init
           ? [vuexInit].concat(options.init)
           : vuexInit
         _init.call(this, options)
       }
     }
   
     /**
      * Vuex init hook, injected into each instances init hooks list.
      */
   
     function vuexInit () {
       const options = this.$options
       // store injection
       if (options.store) {
         this.$store = typeof options.store === 'function'
           ? options.store()
           : options.store
       } else if (options.parent && options.parent.$store) {
         this.$store = options.parent.$store
       }
     }
   }
   
   ```

   在`applyMixin`中,会判断`Vue`的版本,如果是2.0以上版本会将`vuexInit`混入至全局`beforeCreate`钩子中, 否则就重写`Vue`的 `_init`函数,在`_init`函数中调用`vuexInit`,而`vuexInit`本质上只做了一件事,判断当前vue实例中是否有`store`属性,有的话将自身的`$store`属性设置为该值,如果没有则将父组件的`$store`设置为自身的`$store`这样在整个组件树中,所以的组件都是引用的同一个对象,

2. `Store`
