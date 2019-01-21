# 			                            Vuex源码浅析

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

   在`applyMixin`中,会判断`Vue`的版本,如果是2.0以上版本会将`vuexInit`混入至全局`beforeCreate`钩子中, 否则就重写`Vue`的 `_init`函数,在`_init`函数中调用`vuexInit`,而`vuexInit`本质上只做了一件事,判断当前vue实例中是否有`store`属性,有的话将自身的`$store`属性设置为该值,如果没有则将父组件的`$store`设置为自身的`$store`这样在整个组件树中,所有的组件都是引用的同一个对象,

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

   ```javascript
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
   ```

   首先通过解构赋值从参数中提取出插件列表及是否启用严格模式，同时如果未指定模式，则默认关闭严格模式，下面我们创建一个例子来看一看具体`Store`是如何执行的

   ```javascript
   //example.js
   export default new Vuex({
     state:{
         count:1
     },
     actions:{
       addcount({commit},payload){
         commit('ADD_COUNT',payload)
       }
     },
     mutations:{
       'ADD_COUNT'(state,payload){
         payload > 0 ? state.count += payload : state.count++
       }
     },
     modules:{
       state:{
         userName:''
       },
       actions:{
         updateUserName({commit},str){
           commit('SET_USER_NAME',str)
         }
       },
       mutations:{
         'SET_USER_NAME'(state,str){
           if(!typeof str == 'string')return
           state.userName = str
         }
       }
     }
   })
   ```

   我们来看一下`store`的源码如何处理,首先,因为我们没有传入`plugins`和`strict`字段,所以采用默认值,随后创建多个空对象属性,然后将`options`传递给`ModuleCollection`构造函数,

   `ModuleCollection`会以`Module`类为基础构建自身,将当前传入`options`构建为根节点,

   随后调用`installModule(this, state, [], this._modules.root)`,

   ```javascript
   function installModule (store, rootState, path, module, hot) {
     const isRoot = !path.length
     const namespace = store._modules.getNamespace(path)
   
     // register in namespace map
     if (module.namespaced) {
       store._modulesNamespaceMap[namespace] = module
     }
   
     // set state
     if (!isRoot && !hot) {
       const parentState = getNestedState(rootState, path.slice(0, -1))
       const moduleName = path[path.length - 1]
       store._withCommit(() => {
         Vue.set(parentState, moduleName, module.state)
       })
     }
   
     const local = module.context = makeLocalContext(store, namespace, path)
   
     module.forEachMutation((mutation, key) => {
       const namespacedType = namespace + key
       registerMutation(store, namespacedType, mutation, local)
     })
   
     module.forEachAction((action, key) => {
       const type = action.root ? key : namespace + key
       const handler = action.handler || action
       registerAction(store, type, handler, local)
     })
   
     module.forEachGetter((getter, key) => {
       const namespacedType = namespace + key
       registerGetter(store, namespacedType, getter, local)
     })
   
     module.forEachChild((child, key) => {
       installModule(store, rootState, path.concat(key), child, hot)
     })
   }		
   ```

   `installModule`调用中首先会判断当前节点是否存在命名空间配置,如果有则分配命名空间至`_modulesNamespaceMap`,随后将`state`根据`modules`的`key`值注册至根节点;

   随后调用`  const local = module.context = makeLocalContext(store, namespace, path)`

   ```js
   function makeLocalContext (store, namespace, path) {
     const noNamespace = namespace === ''
   
     const local = {
       dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
         const args = unifyObjectStyle(_type, _payload, _options)
         const { payload, options } = args
         let { type } = args
   
         if (!options || !options.root) {
           type = namespace + type
           if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
             console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
             return
           }
         }
   
         return store.dispatch(type, payload)
       },
   
       commit: noNamespace ? store.commit : (_type, _payload, _options) => {
         const args = unifyObjectStyle(_type, _payload, _options)
         const { payload, options } = args
         let { type } = args
   
         if (!options || !options.root) {
           type = namespace + type
           if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
             console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
             return
           }
         }
   
         store.commit(type, payload, options)
       }
     }
   
     // getters and state object must be gotten lazily
     // because they will be changed by vm update
     Object.defineProperties(local, {
       getters: {
         get: noNamespace
           ? () => store.getters
           : () => makeLocalGetters(store, namespace)
       },
       state: {
         get: () => getNestedState(store.state, path)
       }
     })
   
     return local
   }
   ```

   这个方法主要是为了命名空间,首先判断当前`module`是否存在命名空间,如果没有则使用全局`dispatch`和`commit`,如果存在则添加一个错误提醒,随后调用全局`dispatch,commit`,同时如果存在命名空间则只返回该命名空间的`getters`否则返回全局`getters`,之后返回当前模块的`state`值,

   之后对当前模块进行遍历注册`Mutation` `Action` `Getter`

   ```js
   function registerMutation (store, type, handler, local) {
     const entry = store._mutations[type] || (store._mutations[type] = [])
     entry.push(function wrappedMutationHandler (payload) {
       handler.call(store, local.state, payload)
     })
   }
   
   function registerAction (store, type, handler, local) {
     const entry = store._actions[type] || (store._actions[type] = [])
     entry.push(function wrappedActionHandler (payload, cb) {
       let res = handler.call(store, {
         dispatch: local.dispatch,
         commit: local.commit,
         getters: local.getters,
         state: local.state,
         rootGetters: store.getters,
         rootState: store.state
       }, payload, cb)
       if (!isPromise(res)) {
         res = Promise.resolve(res)
       }
       if (store._devtoolHook) {
         return res.catch(err => {
           store._devtoolHook.emit('vuex:error', err)
           throw err
         })
       } else {
         return res
       }
     })
   }
   
   function registerGetter (store, type, rawGetter, local) {
     if (store._wrappedGetters[type]) {
       if (process.env.NODE_ENV !== 'production') {
         console.error(`[vuex] duplicate getter key: ${type}`)
       }
       return
     }
     store._wrappedGetters[type] = function wrappedGetter (store) {
       return rawGetter(
         local.state, // local state
         local.getters, // local getters
         store.state, // root state
         store.getters // root getters
       )
     }
   }
   
   ```

   随后递归的调用`installModule`,

   经过这些处理之后,所有的`State`都会按照`module`存放至根节点的`state`中,所有的`Action` `Mutation`会按照相同名称存放在同一数组的方式存放至`_actions`和`_mutations`,

   随后调用`resetStore`在内部创建一个`Vue`实例,通过`Vue`实现的数据响应来实现其响应化,并且将所有的`getter`注册为`Vue` `computed`属性,以实现根据`state`变化来实时改变派生状态,同时如果是更新`store`会存放前一个实例索引,操作完成后释放旧实例,并将`_vm`指向新生成的`Vue`实例

   ```javascript
   function resetStoreVM (store, state, hot) {
     const oldVm = store._vm
   
     // bind store public getters
     store.getters = {}
     const wrappedGetters = store._wrappedGetters
     const computed = {}
     forEachValue(wrappedGetters, (fn, key) => {
       // use computed to leverage its lazy-caching mechanism
       computed[key] = () => fn(store)
       Object.defineProperty(store.getters, key, {
         get: () => store._vm[key],
         enumerable: true // for local getters
       })
     })
   
     // use a Vue instance to store the state tree
     // suppress warnings just in case the user has added
     // some funky global mixins
     const silent = Vue.config.silent
     Vue.config.silent = true
     store._vm = new Vue({
       data: {
         $$state: state
       },
       computed
     })
     Vue.config.silent = silent
   
     // enable strict mode for new vm
     if (store.strict) {
       enableStrictMode(store)
     }
   
     if (oldVm) {
       if (hot) {
         // dispatch changes in all subscribed watchers
         // to force getter re-evaluation for hot reloading.
         store._withCommit(() => {
           oldVm._data.$$state = null
         })
       }
       Vue.nextTick(() => oldVm.$destroy())
     }
   }
   ```

到这里,基本已经完成了初始化,我们看一下如何去触发;Vuex想要修改`state`必须要通过`Mutation`,但是`Mutation`我们可以选择通过`dispatch`触发`action`去触发,也可以直接`commit`来进行触发,由此来看Vuex中修改`state`主要有两种方法`dispatch`和`commit`

```javascript
const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }
```

在`constructor`中的这段代码,保证在任何情况下调用`dispatch`及`commit`其作用域都为`Store`

首先看一下`dispatch`的实现

```js
  dispatch (_type, _payload) {
    // check object-style dispatch
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown action type: ${type}`)
      }
      return
    }

    this._actionSubscribers.forEach(sub => sub(action, this.state))

    return entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)
  }
```

首先会对参数进行标准化,这在`Vue`及`Vuex`源码中都非常常见(tips:头一次看这种处理的时候真的想说一句,为毛我没想到呢);之后从`_actions`中找到所以以`type`为名注册的`action`,随后遍历调用所有的`subscribeAction`函数,之后调用找到的`action`,

再看一下`commit`的实现,则与`dispatch`大同小异

```js
commit (_type, _payload, _options) {
    // check object-style commit
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    const entry = this._mutations[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown mutation type: ${type}`)
      }
      return
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    this._subscribers.forEach(sub => sub(mutation, this.state))

    if (
      process.env.NODE_ENV !== 'production' &&
      options && options.silent
    ) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
        'Use the filter functionality in the vue-devtools'
      )
    }
  }
```

不同的是,commit会首先调用所有`mutation`随后在调用`_subscribers中`注册的订阅函数,之所以这样是因为`subscribe`订阅函数传入的第二个参数是计算后的`state`,同时`mutations`采用了`_withCommit`来调用则是为了标志是否经由`mutation`修改`state`,如果再严格模式开启时,不通过`_withCommit`函数来调用`mutations`则会报错.

到这里大致的脉络基本就整理完毕了

### 结尾

`VUex`源码虽然代码数据量较少,但是其设计思想及代码都是极其精湛的,我力图写的简单易懂,但是还是有很多地方因篇幅及文笔无法介绍出来或者无法介绍的更好,所以本文只是一个抛砖引玉的作用,希望大家都可以看看`Vuex`的源码,其中的代码思想很值得学习