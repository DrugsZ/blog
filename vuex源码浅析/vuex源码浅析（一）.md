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

   `Store`提供了`Vuex`的核心流程,首先看一下它的`constructor`函数

   ```javascript
     constructor (options = {}) {
       // Auto install if it is not done yet and `window` has `Vue`.
       // To allow users to avoid auto-installation in some cases,
       // this code should be placed here. See #731
       if (!Vue && typeof window !== 'undefined' && window.Vue) {
         install(window.Vue)
       }
   
       if (process.env.NODE_ENV !== 'production') {
         assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
         assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
         assert(this instanceof Store, `store must be called with the new operator.`)
       }
   
       const {
         plugins = [],
         strict = false
       } = options
   
       // store internal state
       this._committing = false
       this._actions = Object.create(null)
       this._actionSubscribers = []
       this._mutations = Object.create(null)
       this._wrappedGetters = Object.create(null)
       this._modules = new ModuleCollection(options)
       this._modulesNamespaceMap = Object.create(null)
       this._subscribers = []
       this._watcherVM = new Vue()
   
       // bind commit and dispatch to self
       const store = this
       const { dispatch, commit } = this
       this.dispatch = function boundDispatch (type, payload) {
         return dispatch.call(store, type, payload)
       }
       this.commit = function boundCommit (type, payload, options) {
         return commit.call(store, type, payload, options)
       }
   
       // strict mode
       this.strict = strict
   
       const state = this._modules.root.state
   
       // init root module.
       // this also recursively registers all sub-modules
       // and collects all module getters inside this._wrappedGetters
       installModule(this, state, [], this._modules.root)
   
       // initialize the store vm, which is responsible for the reactivity
       // (also registers _wrappedGetters as computed properties)
       resetStoreVM(this, state)
   
       // apply plugins
       plugins.forEach(plugin => plugin(this))
   
       const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
       if (useDevtools) {
         devtoolPlugin(this)
       }
     }
   ```

   在前面的install中我们提到,`Store`文件中维护了一个私有变量`Vue`来判断是否安装过,在`Store`构造函数中同样进行了判断,如果`Vue`变量为空,则进行安装流程,之后进行了三次断言,分别判断是否安装,是否存在`Promise`,当前实例是否通过`new`操作符构造,

   首先我们看一下`options`的结构

   > state:Vuex store 实例的根 state 对象。
   >
   > mutations：在 store 上注册 mutation，处理函数总是接受 `state` 作为第一个参数
   >
   > actions：在 store 上注册 action。处理函数总是接受 `context` 作为第一个参数，
   >
   > getters:在 store 上注册 getter，
   >
   > modules:包含了子模块的对象，会被合并到 store
   >
   > plugins:一个数组，包含应用在 store 上的插件方法。这些插件直接接收 store 作为唯一参数，可以监听 	   mutation或者提交 mutation
   >
   > strict:使 Vuex store 进入严格模式，在严格模式下，任何 mutation 处理函数以外修改 Vuex state 都会抛出错误。
   >
   > devtools:为某个特定的 Vuex 实例打开或关闭 devtools。

   好，我们来看看vuex是如何使用这些属性的:
