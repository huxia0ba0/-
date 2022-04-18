#### Vue打包后不同版本的vue
    
    
    完整版：同时包含编译器和运行时的版本。
    编译器：用来模板字符串编译成为JavaScript渲染函数的代码，体积小、效率高。基本上就是出去编译器的代码 (将模板转译render函数)。
    UMD：UMD版本通用的模板版本，支持多种模块方式。vue.js默认文件就是运行时+编译器的UMD版本CommonJs(cjs)：CommonJS版本用来配合老的打包工具比如Browserify或webpack1
    ESModule：从2.6开始Vue会提供两个ESModules(ESM)构建文件，为现代打包工具提供的版本
        ESM格式被设计为可以被静态分析，所以打包工具可以利用这一点来进行"tree-shaking"并将用不到的代码排出最终的包。

``` html
<body>
    <div id="app">
        Hello World
    </div>
</body>
```

 **vue完整版（可以解析template模板）**
 
``` javascript
    <script src="./vue.js"></script>
    <script>
        const vm = new Vue({
            el: '#app',
            template: '<h1>{{ msg }}</h1>',
            data: {
                msg: 'hello vue'
            }
        })
    </script>
    
    // html页面展示内容为 替换app容器的 h1标签 <body><h1>hello vue</h1></body>
```

** vue运行时（不能解析template模板，只能解析render函数）**

``` javascript
    <script src="./vue.runtime.js"></script>
    <script>
        const vm = new Vue({
            el: '#app',
            render(h) {
                return h('h1', this.msg )  
            },
            data: {
                msg: 'hello vue'
            }
        })
    </script>
    
    // html页面展示内容为 替换app容器的 h1标签 <body><h1>hello vue</h1></body>

```

#### 入口文件

1.在package.json文件中 script命令 "dev": "rollup -w -c script/config.js"

2.找到script文件夹的config文件

3.config文件最后导出getConfig函数(参数为TARGET环境变量)生成rollup配置文件


**当同时传入render函数和template模板**

1. 在compiler文件中的$mount方法中会处理 el(创建vue实力传入的选项)

2.通过query函数处理el
``` javascript
    // el不能是 body或html，否则会警告，直接返回this
function query(el) {
    // el不是字符的话 el为dom对象，否则为选择器
   if (typeof el === 'string') {
        // 根据el找到选择器
        const selected = documnet.querySelector(el)
        // 没有找到就报错，找到返回选择器
    if (!selected) {
        process.env.NODE_ENV !== 'production' && warn('Cannot find element:') + el
    }
    return documnet.createElement('div')
    return selected
   } else {
       return el
   }
}

```

3.获取options选项 const options = this.$options,判断options选项是否有render选项，如果没有把template模板转换为render函数

4.调用mount方法渲染dom

#### Vue初始化的过程

四个导出Vue的模块
1. src/platforms/web/entry-runtime-with-compiler.js

    1.1web平台相关入口
    1.2重写平台相关$mount方法(内部可以将template转为render)
    1.3注册了Vue.compile方法，传递一个HTML字符串返回render函数

2. src/platforms/web/runtime/index.js
    2.1web平台相关
    2.2注册和平台相关的全局指令:v-model、v-show
    2.3注册和平台相关的全局组件:v-transition、v-transition-group
    2.4全局方法
        2.4.1__patch__:把虚拟DOM转换为真实DOM
        2.4.2$mount:挂载方法

3.src/core/index.js
    3.1与平台无关
    3.2设置了Vue的静态方法,initGlobalAPI(Vue)

4.scr/core、instance/index.js
    4.1与平台无关
    4.2定义了构造函数，调用了this._init(options)方法
    4.3给Vue中混入了常用的实例成员

##### 初始化问题
1.语法红线
解决：文件->首选项->设置->找到"javascript.validate.enable" 设置为false(不检查js语法问题，防止flow报错， 使用flow语法)

2.使用flow泛型，后面的代码会丢失高亮显示
解决：下载babel-javascript插件
问题：代码功能丢失 例：control路径无法跳转

#### 静态成员初始化
src/core/index.js -> initGlobalAPI

##### 2.注册Vue.mixin实现混入

``` javascript
Vue.mixin = function (mixin) {
    // 此时this是Vue，给options赋值mixin 注册全局的mixin
    // mergeOption作用是把mixin拷贝给this.options
    this.options = mergeOptions(this.options, mixin)
    return this
}
```

##### 3.注册Vue.extend 基于传入的options返回一个组件的构造函数

``` javascript 
Vue.extend = function () {
    // 所有的vue组件都继承于vue
}
```

##### 4.initAssetRegisters(Vue) 注册Vue.directive、Vue.component、Vue.filter

``` javascript
    function initAssetRegisters(Vue) {
    // 遍历ASSET_TYPES数组，为Vue定义相应方法
    // ASSET_TYPES包括了directive、component、filter
        ASSET_TYPES.forEach(type => {
            Vue[type] = function (id, definition) {
            // 如果没有传定义的而函数或方法，会去vue的options上找到对应id（名字）的组件/指令.. 并返回
                if (!definition) {
                    return this.options[type + 's'][id]
                }
                
                // 如果类型为组件，判断definition是否为原始对象(原型是否为[object, object])
                // 如果传的definition 为函数的话 会在最下面直接存储在options中
                if (type === 'component' && isPlainObject(definition) {
                definition.name = definition.name || id
                // this.options._base为 Vue 把组件配置转换为组件的构造函数
                definition = this.options._base.extend(definition)
                }
                
                // 如果类型为指令
                // 如果传的definition 为函数的话 会在最下面直接存储在options中
                if(type === 'directive' && typeof definition === 'function') {
                    definition = { definition = {bind: definition, update: definition }
                }
                
                // 全局注册存储资源并赋值
                this.options[type + 's'][id] = definition
                return definition 
            }
        )
    }
```

#### 实例成员初始化

src/core/instance/index.js

##### 1.initMixin  注册vm 的_init方法，初始化vm
// 给Vue实例的原型上添加一个init方法

##### 2.stateMixin  注册vm的$data $props $set $delete $watch

// 给$date、$props属性赋值，开发环境不允许设置他们的值

// 在原型上设置了$set
// 在原型上设置了$delete
// 在原型上设置了$watch

##### 3.eventsMixin 初始化事件相关方法

// 在原型上设置了 $on  (注册事件)

    判断参数是否为数组，为数组的话，循环遍历数组继续调用$on方法
    不为数组的话，在vm._event上找到对应名称的事件数组，如果没有就设置为空数组，并把处理函数推进数组
    else {
        (vm._event[event] || (vm._event[event] = [])).push(fn)
    } 
    
// 在原型上设置了 $once (注册事件, 只触发一次)

// 在原型上设置了 $off (取消事件) 

// 在原型上设置了 $emit (触发事件) 


##### 4.lifecycleMixin(Vue)  
    // _update 方法的作用是把Vnode渲染成真实DOM
    // 首次渲染会调用，数据更新会调用
``` javascrpt   
    function lifecycleMixin () {
        Vue.prototype._uodate = function () {
            // 最核心的是
            // 将虚拟dom转换为真实dom 挂在到$el
            vm.$el = vm.__pathch__()
        }
        
        // 强制更新
        Vue.prototype.$forceUpdate = function () {
            
        }
        
        Vue.prototype.$destroy = function () {
            
        }
    }
```

##### 5.renderMixin() 混入render $nextTick _render

// 安装渲染的一些帮助方法
installRenderHelpers(Vue.prototype)

// 注册$nextTick
Vue.prototype.$nextTick

// 注册_render
Vue.prototype._render

##### Vue初始化实例成员int

Vue初始化时会调用_init方法
_init方法在 initMixin中注册

initMixin方法后会在Vue原型上挂载_init
```javascript
    // 1.合并用户传入的options和vue构造函数的options
    
     //$children $parnet $root $refs 
     2.initLifecycle   
     
     // vm的事件监听初始化， 将父组件附加的事件绑定在当前组件 
     3.initEvents
     
     // vm的编译render初始化
     // $slots $scopedSlot _c $createElement $attrs $listeners
     
     _c和$createElement的区别
     // 对手写render函数进行渲染的方法
     // vm.$createElement =(a, b, c, d) => createElement(vm, a, b, c, d. true)
     
     // 模板编译生成的render进行渲染的方法
     // vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
     4.initRender

     // 生命钩子的回调 beforeCreate
     5.callHook(vm, 'beforeCreate')
     
     // 取出vm $options.inject ，遍历注入vue实例设置响应式数据，循环inject，判断_provide中是否包此属性，如果包含就存储起来并返回
     6.initInjections 
     
     // 取出vm $options中的provide存储到this._provide
     7.initProvide
     
     8.callHook(vm, 'created')
     
     // 获取vm.$options
     // initProps 把props成员转换为响应书数据，并注入vue实例
     // initMethods warning提示三项 必须是function 不可以设置props同名 不建议使用$ _开头命名
     // 将methods注入vue实例(会判断非函数传入noop空函数，函数会bind改变this为vm)
     // initData 判断有无data 如果没有调用observe将vm._data设置为空对象，如果有调用initData
     initData 判断data是否为函数 如果是调用getData(通过call(vm, vm)调用data),如果不是就直接设置为data
     
     // 获取props methods判断是否重名
     // 如果以_ $开头不会注入vue实例, 不以这两项开头的话会调用proxy注入vue实例
     
     // 将data设置为响应式对象
     
     
     // initComputed注入计算属性
     // initWatch注入监听器 
     
     // 初始化vm的_props/methods/_data/computed/watch
     9.initState
     
     
     

      // 获取this.$options
      // 判断有无$options.render属性
      if (render) {
         如果没有判断template
          if (template) {
              // 判断模板第一个字符是#,模板是否是id选择器
              if (template.charAt(0) === '#' {
                  // 获取对应id选择器dom的innerHTML
                  template = idTotemplate(template)
                  
                  // 如果有nodeType元素证明他是个dom元素，此时直接返回innerHTNL
              } else if (template.nodeType) {
                  template = template.innerHTML
              }
          } 
          // 如果没有template，并且有el， 那么获取el 的outHTML作为模板
          else if(el) {
              // getOuterHTML 会判断有无outerHTML，有返回，无的话创建DOM将el的Node元素appendChild中
              template = getOuterHTML(el)
          }
          
          // 通过compileToFunctions将template转化为render函数
          
         render生成后存储在options中
          options.render = render
          
          // 调用mount方法 渲染dom
          
          mount方法中调用mountComponent
          
          调用callHook(vm, 'beforeMount')
          
          定义updateComponent函数更新dom
          
          创建 new Watcher
          调用callHook(vm, 'mounted')
          
          
      }
     
     10.vm.$mount(vm.$options.el)
     
     
```

##### 数据响应式原理

###### 响应式入口
src\core\instance\init.js
    调用initState(vm) vm状态的初始化
    初始化了 _data、_props、methods等

src\core\instance\state.js

```javascript  
// 数据的初始化
if (opts.data) {
    iniData(vm)   //  最后会调用observe
} else {
    observe(vm._data = {}, true)
}
```

###### Observe
```javascript
export function observe (value, asRootData) {
     // 1.判断value 是否是对象
    
    // 2.如果value有__ob__属性 代表ob做过响应式处理，  取出直接返回
    if (hansOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
        ob = value.__ob__
    } else if(/*判断是否可以响应式处理*/) {
        // 创建Observer
        ob = new Observer(value)
    }
    
}

// observer 对数组，对象进项响应式处理

export class Observer {
    // 观测对象
    value: any;
    // 依赖对象
    dep: Dep;
    // 实例计数器
    vmCount: number;
    
    constructor (value){
        this.value = value
        this.dep = new Dep()
        this.vmCount = 0;
        // 将实例挂在到观察对象的__ob__属性
        def(value, '__ob__', this)
        // 数组的响应式处理
        if (Array.isArray(value)) {
            // 判断是否有__proto__, 对象是否包含该属性(处理兼容性问题)
            if (hasProto) {
            
             // arrayMthods->数组方法
            // 内部遍历数组方法
            
           methodsToPatch.forEach(function (method) => {
                 // 对于push,unshift,splice,新插入的值，会重新遍历数组对于对象设置为响应式数据
                 if (inserted) ob.observeArray(inserted)
                 // 调用修改数组方法后，调用数组ob对象发送通知 更新视图
                 ob.dep.notify()
                 
                 // 所以数组无法针对于length 索引修改，为视图更新做出反应
            })
          
            
            // protoAugement 方法 是将 target.__proto__ 赋值为第二个参数，即value.__proto__ = arrayMthods(重写数组方法)
            
             protoAument(value, arrayMthods)
            } else {
                copyAugment(value, arrayMethods, arrayKeys)
            }
            // 为数组中的每一个对象创建一个observer实例
            this.observeArray(value)
        } else {
            // 遍历对象中的每一个属性，转换为setter / getter
            this.walk(value)
        }
    }
    
}
```

###### defineReactive

```javascript
   export function defineReactive(
    obj,
    key,
    val,
    customSetter?,
    shallow?,
   ) {
       // 创建依赖对象实例
       const dep = new Dep()
       // 获取 obj 的 属性描述符对象
       const property = Object.getOwnPropertyDescriptor(obj, key)
       // 当前属性不可配置(意味着不可以通过defineProperty配置get set)
       if (property && property.configurable === false) {
           return 
       }
       
       // 取出用户设置的getter setter
       const getter = property && property.get
       const setter = property && property.set
       
        // 判断是否递归观察子对象，并且子对象都转换成getter/setter, 返回子观察对象
        let childOb = !shallow && observe(val)
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: function reactiveGetter () {
            // 判断用户是否设置了getter，如果有则设置用户的值，没有就设置传入的val
                const value = getter ? getter.call(obj) : val
             // 如果存在当前依赖目标，即watcher对象，则建立依赖
             if (Dep.target) {
                 dep.depend() 
                 // 如果子观察目标存在，建立子对象的依赖关系
                 if (childOb) {
                     childOb.dep.depend()
                     // 如果属性是数组，则特殊处理收集数组对象依赖
                     if (Array.isArray(value)) {
                         dependArray(value)
                     }
                 }
             }
                
               return value
            }，
            set: function reactiveSetter(newVal) {
                const value = getter ? getter.call(obj) : val
                // 判断 新旧是否相同， ||后是判断NaN情况
                if (newVal === value || (newVal !== newVal && value !== value)) {
                    return 
                }
                
                // 如果setter不存在直接返回，只读
                if (getter && !setter) return 
                //
                if (setter) {
                    setter.call(obj, newVal)
                // getter setter都不存在,直接将newVal赋值
                } else {
                    val = newVal
                }
                // 如果新值newVal是对象，将新对象的属性转换getter setter
                childOb = !shallow && observe(newVal)
                // 派发更新(发布更改通知)
                dep.nnotify()
            }
        })
   }
```

###### 依赖收集

```javascript
 // 如果存在当前依赖目标，即watcher对象，则建立依赖             
             // 会在访问属性会收集依赖即watcher对象
             if (Dep.target) {
                // depend方法会把watcher对象push到subs数组中
                 dep.depend() 
                 // 如果子观察目标存在，建立子对象的依赖关系
                 
                 // 判断子对象的observe
                 if (childOb) {
                    // 为子对象收集依赖
                     childOb.dep.depend()
                     // 如果属性是数组，则特殊处理收集数组对象依赖
                     if (Array.isArray(value)) {
                         dependArray(value)
                     }
                 }
             }
```

###### Watcher类

watcher 分为三类  1.计算属性computed， 2.侦听器Watcher， 3.渲染Watcher

watcher的创建顺序 1 ->  2 ->  3

渲染Watcher的创建时机
/src/core/instance/lifecycle.js
```javascript
    export function mountComponent () {
        ...
        
        new Watcher(vm, undateComponent, noop, {
            // 触发beforeUpdate
            before () {
                if (vm._isMounted && !vm._isDestroyed) {
                    callHook(vm, 'beforeUpdate')
                }
            }
        }, true /* 标识是渲染watcher  isRenderWatcher */) {
             
        }
    }
    
    
// Watcher类内部
constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
)
    if (isRenderWatcher) {
     // 如果是渲染watcher就记录到vm的_watcher
        vm._watcher = this
    }
    // 将所有的watcher记录到vm的_watchers(记录所有watcher)
    vm._watchers.push(this) 
    
    if (typeof expOrFn === 'function') {
        this.getter = expOrFn
    } else {
        // expOrFn 是字符串的时候，例如watch { 'person.name': function ...}
        // parsePath('person.name')返回一个函数获取person.name的值
        this.getter = parsePath(expOrFn)
        
    }
    
    
    //options没有传递的话  渲染watcher 里面lazy 一般是false会立即执行get方法
    // 计算属性watcher中 lazy会是true 延时执行
    this.value = this.lazy ? undefined : this.get()
    
    
    
    // get方法
    get () {
        pushTarget(this)
        // this.getter 渲染watcher存储的就是updateComponent，执行完updateComponent，我们就把虚拟dom转换为真实dom并渲染到页面 
        value = this.getter.call(vm, vm)
        
        // 执行完毕后出栈
        popTarget()
        // 会把当前watcher 从Dep subs数组移除，会把watcher记录的dep移除
        this.cleanupDeps)()
    }
    
    Dep.target = null
    const targetStack = []
    // 入栈并将当前watcher 赋值给Dep.target
    // 父子组件嵌套的时候先把父组件对应的watcher入栈
    // 再去处理子组件的watcher，子组件的处理完毕后，再把父组件对应的watcher出栈继续操作
    // pushTarget
    function pushTarget (target) {
        targetStack.push(target)
       
        Dep.target = target
    }
    
    // popTarget
    function popTarget () {
        // 出栈操作
        targetStack.pop()
        Dep.target = targetStack[targetStack.length - 1]
    }
```

```javascript
数据更新时 watcher 的操作
/src/core/observer/dep.js

// 数据发生改变时会调用dep的notify方法
notify() {
    // 克隆subs数组
    const subs = this.subs.slice()
    
    // 按照id顺序排列确保按watcher创建顺序执行
    subs.sort((a, b) => a.id - b.id)
    
    // 调用每个订阅者的update方法实现更新
    for (let i = 0, i = subs.length; i < 1; i++) {
        subs[i].update()
    }
}

 // update方法
function update () {
    // 渲染watcher lazy sync 一般都是false 会执行queueWatcher
    if (this.lazy) {
        this.dirty = true
    } else if (this.sync) {
        this.run()
    } else {
        queueWatcher(this)
    }
}

// queueWatcher方法(把当前watcher放入一个队列)

function queueWatcher (watcher) {
     const id = watcher.id
     // 如果has中id 为null，代表此watcher未处理
     // 此操作防止watcher被重复处理    
     if (has[id] == null) {
        // 标记为已处理，往下处理该watcher
         has[id] = true
         // flushing为true 代表queue正在被处理
         if (!flushing) {
             // 没有被处理就把watcher放在queue队列
             queue.push(watcher)
         } else {
            // 正在被处理就要找到合适的位置放进来
            let i = queue.length - 1
            //i为队列的元素个数 index为处理到第几个元素
            // i > index表示队列还没有被处理完
            // 从后往前取队列的id 判断是否大于待处理watcher的id，如果大于那么这个位置就是我们要插入的位置
            while(i > index && queue[i].id > watcher.id) {
                i--
            }
            // 放入到指定的位置
            queue.splice(i + 1, 0, watcher)
         }
         
         // waiting 为false代表没有被执行呢
         if (!waiting) {
             waiting = true
             // 开发环境直接调用flushSchedulerQueue
             if （process.env.NODE_ENV !=='production' && !config.async) {
                 flushSchedulerQueue()
                 return
             }
             // 生产环境在nextTick调用flush函数
             nextTick(flushSchedulerQueue)
         }
     }
}

// flushSchedulerQueue函数
 function flushSchedulerQueue () {
     // 标记正在处理watcher队列
     flushing = true
     // queue从小到大排序 -> 也就是按照创建顺序排列下面为原因
     // 1.保证组件创建顺序为父-子
     // 2.组件的用户watcher要在对应的渲染watcher之前运行
     // 3.如果一个组件在他的父组件执行前被销毁了，此watcher应该跳过
     queue.sort((a, b) => a.id - b.id)
     
     // 不要缓存length 因为在执行过程中queue有可能添加新的watcher
     for (index = 0; index < queue.length; index++) {
         watcher = queue[index]
         // before是为了处理beforeUpdate钩子函数
         if (watcher.before) {
             watcher.before()
         }
         // 此watcher已被处理，把该id设为null，目的是数据发生变化后该watcher还能继续运行
         id = watcher.id
         has[id] = null
         // 最终调用run方法
         watcher.run()
     }
 }


// run方法
function run () {
    // 此watcher是否为存活的状态
    if (this.actvie) {
        // 调用get方法，也就是执行updateComponent渲染组件更新视图(上面有记录该方法)
        
        // 渲染watcher没有返回值
        const value = this.get()
        
        if (value !== this.value || isObject(value) || this.deep
        ) {
            // 记录旧值
            const oldValue = this.value
            // 记录新值
            this.vaue = value
            this.value = value
            // 此下是用户watcher处理 
            if (this.user) {
                // 调用回调函数
               try {
                    this.cb.call(this.vm, value, oldVaue)
               } catch(e) {
                   // 防止用户watcher可能回调会异常
               }
            } else {
                // 不是用户watcher会直接调用cb
                
                // 渲染cb传入的是noop
                this.cb.call(this.vm, value, oldValue)
            }
        }
    }
}


整体为当数据发生变化后，调用dep的notify方法，然后遍历subs队列调用update方法，update方法内部会调用queueWatcher方法，queueWatcher方法内部会判断该watcher是否被执行，没有执行的话会将该watcher放入queue队列里，然后执行flushSchedulerQueue方法，flushSchedulerQueue方法内部遍历queue队列，循环调用watcher的before方法(beforeUpdate钩子)，将此处理完的watcher的id设置为null为下次视图变化该数据能正常执行，然后执行每个watcher的run方法，run方法中调用了get方法，get方法内部调用了pushTarget最终调用getter方法，也就是执行updateComponent方法来更新视图
```

###### set方法（动态添加一个响应式属性， 确保更新视图）

```javascript
    obj没有name属性，直接赋值此时不会更新视图
    vm.$set(vm.obj, 'name', 'zs')
    
    // $set通过索引修改数组
    vm.$set(vm.arr, 0, 100)
    
    
    
    
    function det (target, key, val) {
        // 判断target是否是数组，key是否是合法的索引
        if (Array.isAarry(target) && isValidArrayIndex(key)) {
        // 判断数组length和设置的属性哪个数值大，则设置谁为length
            target.length = Math.max(target.length, key)
        // splice 在array进行了响应化的处理
        target.splice(key, 1, val)
        return value
        }
        
        // 如果 key 在对象中已经存在直接赋值并且不是原型上的成员
        if (key in target && !(key in object.prootype)) {
            target[key] = val
            return val
        }
        
        // 获取target中的observer对象
        const ob = target.__ob__
        // 如果target是vue实例或者$data 直接返回
        if (target.isVue || (ob && ob.vmCount)) {
            // warn(....)
            return val
        }
        // 如果ob不存在，target不是响应式对象 直接赋值
        if (!ob) {
            target[key] = val
            return val
        }
        
        // 把key设置为响应式属性
        defineReactive(ob.value, key, val)
        // 发送通知
        ob.dep.notify()
        return val
    }
```

###### delete方法（动态删除一个响应式属性， 确保更新视图）

```javascript

    vm.$delete(vm.obj, 'msg')
    
    
    function del (target, key) {
            // 判断target是否是数组，key是否是合法的索引
        if (Array.isAarry(target) && isValidArrayIndex(key)) {
        // splice 在array进行了响应化的处理
        target.splice(key, 1)
        return value
        }
        
         // 获取target中的observer对象
        const ob = target.__ob__
        
           // 如果target是vue实例或者$data 直接返回
        if (target.isVue || (ob && ob.vmCount)) {
            // warn(....)
            return val
        }
        // 如果target对象没有key属性 直接返回
        if (!hasOwn(target, key)) {
            return
        }
        
        // 删除属性
        delete target[key]
        // 没有ob属性说明不是响应式数据 直接返回
        if (!ob) {
            return 
        }
        // 通过ob 发送通知
        ob.dep.notify()
    } 
```


###### watcher

```javascript
    // 侦听器
    // croe/instance/state.js
    
    export function initState(vm) {
        // 创建侦听器(vm对象， 用户配置的watch选项)
        initWatch(vm, opts.watch)
    }



    function initWatch (vm, watch) {
        // 遍历创建的所有watch
        for (const key in watch) {
            const handler = watch[key]
            // handler处理对象可以是一个数组
            // 如果处理函数为数组那么就遍历数组处理
            if (Array.isArray(handler)) {
                for (let i = 0; i < handler.length ; i++) {
                    createWatcher(vm, key, handler[i])
                }
            } else {
                createWatcher(vm, key, handler)
            }
        }
    }
    
    function createWatcher (vm, expOrFn. handler, options) {
        // 如果处理对象是对象形式,就取出handler
        if (isPlainObject(handler)) {
            options = handler
            handler = handler.handler
        }
        
        // 如果处理方法是字符串 就在vm实例里面去找对应的方法
        if (typeof handler === 'string') {
            handler = vm[handler]
        }
        
        return vm.$watch(expOrFn, handler, options)
    }
    
    
    $watch = function (expOrFn, cb, options) {
        // 获取Vue实例
        const vm = this
        // 判断cb 是否为对象
        if (isPlainObject(cb)) {
            return createWatcher(vm, expOrFn, options)
        }
        
        options = options || {}
        // 标记用户 watcher
        options.user = true
        // 创建用户watcher对象
        const watcher = new Watcher(vm, expOrFn, cb, options)
        // 判断immediate是否为true
        if (options.immediate) {
            try {
                cb.call(vm, watcher.value)
            } catch (err) {
                
            }
        }
        // 返回取消监听的方法
        return function unwatchFn () {
            watcher.teardown()
        }
    }
```

###### nextTick
```javascript
    // core/instance/index.js
    // renderMixin(Vue)
    
    function nextTick(cb, ctx) {
        let _resolve
        // 把cb加上异常处理存入 callbacks数组中
        callbacks.push(() => {
            if (cb) {
                cb.call(ctx)
            } else if (_resolve(ctx)) {
                _resolve(ctx)
            }
        })
        // 标记队列正在处理
        if (!pending) {
            pending = true
            // 调用
            timerFunc()
        }
    }
    
    // 核心 timerFunc
    const p = Promise.resolve()
    timerFunc = () => {
        // 利用promise执行微任务 所有同步任务执行完后， 再执行
        p.then(flushCallbacks)
    }
    
    // 如果浏览器不支持微任务时 会降级为宏任务
    
    function flushCallbacks () {
        pending = false
        // 备份copies数组
        const copies = callbacks.slice(0)
        callbacks.length = 0
        for (let i = 0; i < copies.length; i++) {
            // 循环调用
            copies[i]()
        }
    }
     1.将nextTick中的处理函数放入一个callbacks数组中
     2.调用timerFunc中通过微任务去调用每一个处理函数
     3.里面会根据浏览器兼容问题判断去使用promise 或者 settimeout
    // nexttick获取dom最新的数据
    // 微任务执行时 dom元素还没有渲染到浏览器上
    // nexttick回调函数执行前 数据已经被改变了，数据改变后通知watcher渲染视图，watcher首先做的是把dom更新(更新dom树)，dom更新到浏览器上是当前事件执行之后。此时微任务是从dom树上获取到的数据，此时dom还没有渲染到浏览器上
    
   
```


#### 虚拟DOM

###### h函数

1.也就是vm.$createElement(tag, data, children， normalizeChilren)

tag 标签名称或者组件对象
data 描述tag，可以设置DOM的属性或者标签的属性
children tag中的文本内容或者子节点

2.h函数返回结果为虚拟domVNode

###### VNode
VNode的核心属性
tag  data  children text  elm(真实dom)  key



###### createElement VNode的创建过程

core/instance/lifecycle.js

```javascript
    function mountComponent () {
        updateComponent = () => {
            // _render生成VNode
            // _update生成真实DOM
           vm._update(vm._render(), hydrating)
        }
        
        new Watcher(vm, updateComponetn, noop, {
            before () {
            callHoo
            }
        })
    }
    
    
    
core/instance/render.js
function renderMixin (Vue) {
    Vue.protoType._render = function () {
    const { render } = vm.$options
    // vm._renderProxy 相当于vue实例
    //  vm.$createElement相当于h函数
        vnode =
        render.call(vm._renderProxy, vm.$createElement)
    }
}

// 编译生成的render进行渲染
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)

// render函数由用户传入进行渲染
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

function createElement (
    context: vue实例
) {
     // createElement函数处理参数
     
     return _createElement()
}

function _createElement () {
    // 传入的tag是字符串
   if (typeof tag === 'string') {
        // 判断tag参数是否是html的保留标签
    if (config.isReservedTag(tag)) {
        // 创建Vnode
        vnode = new Vnode()
        // 判断是否是自定义组件
    } else if ((!data || !data.pre)) {
        vnode = createComponent
        // 如果这些都不是那么他就是自定义标签
    } else {
        // 创建自定义标签 vnode节点
        vnode = new VNode()
    }
    // tag不是字符串 那么他应该是组件
   } else {
       vnode = createComponent()
   }
   
   return vnode
}
```

###### update VNode的处理过程
core/instance/lifecycle.js
```javascript

Vue.prototype._update = function (vnode, hydrating) {

    const prevVnode = vm._vnode
    // !prevNode 代表是首次渲染
    if(!prevVnode) {
        vm.$el = vm.__patch__(vm.$el, node, hydratomh, false)
    } else {
        vm.$el = vm.__patch__(prevVnode, vnode)
    }
}
```

lifecycle文件中
mountComponent函数中调用updateComponent
updateComponent中调用了vm._update(vm._render(), hydrating)
vm._render()中调用createElement函数创建Vnode

vm._update中调用 patch函数对比新旧dom把更新后的dom存储到vm.$el中     


######patch方法

patch初始化
```javascript
    src/platforms/web/runtime/index.js
    
    import { patch } from "./patch"
    Vue.prototype.__patch__ = inBrowser ? patch : noop
    
    
    patch函数
    src/platforms/web/runtime/patch.js
    // 1.nodeOps是对dom操作的一些api
    // 2.modules modules = platformModules.concat(baseModules)
    // 3.platformModules和平台相关的生命周期函数
    // 4.baseModules处理指令和ref
    const patch = createPatchFunction({
       nodeOps, modules  
    })
    
   core/vdom/patch 
   createPatchFunction函数
   
   function createPatchFunction (backend) {
      const cbs = {}
      // 遍历hooks钩子函数
      for (i = 0; i < hooks.length; ++i) {
          cbs[hooks[i]] = []
          for (j = 0; j < modules.length; ++j) {
              if (isDef(modules[j][hook[i]])) {
              // cbs属性为钩子函数
              // 对应值存储钩子提供的函数
                 // cbs['update'] = [updateAttrs, updateClass, ...] cbs[hooks[i].push(modules[j][hooks[i]])]
              }
          }
      }
      
      return function patch()
   }
    
    
```

###### patch函数执行过程
```javascript
    function patch(oldVnode, vnode, hydrating, removeOnly) {
        // 新的vnode 不存在
        if (isUndef(vnode)) {
            // 老的Vnode存在 执行Destroy函数
            if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
            return
        }
        // 存储新插入vnode节点，目的是这些vnode对应的节点挂载到dom上后触发insert钩子函数
        const inserteVnodeQueue = []
        const isInitialPatch = false
        
        // 老的Vnode不存在
        // 情况在mount方法没有传参数，只是创建了该组件DOM，但没有挂载到dom树
        if(isUndef(oldVnode)) {
            // 转换为真实DOM
            createElm(vnode, insertedVnodeQueue)
            isInitialPatch = true
            // 新的老的Vnode都存在
        } else {
            // 判断老Vnode是否有notetype属性，有的话代表为真实DOM，也就是首次渲染
            const isRealElement = isDef(nodeVnode.nodeType)
            
            if (!isRealElement && sameVnode(oldVnode, vnode) {
                // 更新操作 diff算法
                // 对比新旧vnode 更新差异
                patchVnode(oldVnode, vnode, insertedVnodeQueue)
            } else {
                // 老Vnode是真实DOM,也就是首次渲染
                if (isRealElement) {
                    // emptyNodeAt是把真实DOM转换为Vnode
                    oldVnode = emptyNodeAt(oldVnode)
                }
                
                const oldElm = oldVnode.elm
                const parentElm = nodeOps.parentNode(oldElm)
                // createElm把vnode转换真实DOM挂载到parentELm
                createElm(vnode, insertedVnodeQueue, pldElm.leaveCb ? null : parentElm, nodeOps.nextSibling(oldElm))
                
                if (isDef(parentElm)) {
                    // 移除oldVnode,并且触发钩子
                    // removeVnode会判断oldvnode文本节点还是标签节点，标签节点从dom上移除并触发remove钩子函数，触发destory函数， 如果是文本节点直接在DOM树上删除
                    removeVnodes([oldVnode], 0, 0)
                    
                    // 如果parentElm不存在说明oldVnode不存在dom树上
                    // 此时判断oldVnode有无tag属性 有的话调用destory钩子
                } else if (isDef(oldVnode.tag)) {
                    invokeDestroyHook(oldVnode)
                }
            }
        }
        // isInitialPatch 代表该节点是否挂载在dom树上，如果没有挂载dom树上是不会触发insertedVnodeQueue里面的钩子函数
        // 触发insertedVnodeQueue队里中钩子函数insert
        invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
        return vnode.elm
    }
 
 
 ###### createElm  vnode->真实dom->挂载到dom树->触发钩子函数
    
    function createElm (
     vnode,
     insertedVnodeQueue,
     parentElm, // 挂载到的父节点
     nodeOps.nextSibling(oldElm) 
    ) {
            // vnode.elm存在的话 说明有真实dom渲染过
            // ownerArray 代表有子节点
         if (isDef(vnode.elm) && isDef(ownerArray)) {
            // 克隆一份
             vnode = wonerArray[index] = cloneVNode(vnode)
             
             const data = vnode.data
             const children = vnode.children
             const tag = vnode.tag
             // 1.vnode是标签
             if (isDef(tag)) {
                // 判断vnode是否有ns命名空间 实际上是出svg情况
                 vnode.elm = vnode.ns ? nodeOps.createElementNS(vnode.ns, tag) : nodeOps.createElement(tag, vnode)
                 // setScope为vnode所对应的dom元素设置作用域，并且设置scopeId
                 setScope(vnode)
                 // 把子元素转换为DOM对象
                 // createChildren函数在下面
                 createChilren(vnode, children, insertedVnodeQueue)
                 if (isDef(data)) {
                    // 触发create钩子函数
                    invokeCreateHooks(vnode, insertedVnodeQueue)
                 }
                 // 插入到parentElm
                 // insert函数在下面
                 insert(parentElm, vnode.elm, refElm)
             // 2.判断vnode是注释节点
             } else if(isTrue(vnode.isComment)) {
                 // 利用createComment创建注释节点存入elm，并且插入dom树
                  vnode.elm = nodeOps.createComment(vnode.text)
                  insert(parentElm, vnode.elm, refElm)
             
             // 3.判断是文本节点  
             } else {
                  // 利用createTextNode创建文本节点存入elm，并且插入DOM
                 vnode.elm = nodeOps.createTextNode(vnode.text)
                 insert(parentElm, vnode.elm, refElm)
             }
         }
    }
    
    function createChilren (vnode, children, insertedVnodeQueue) {
        if (Array.isArray(children)) {
          for(let i = 0; i < children.length; i++) {
            // 再调用createElm转换真实DOM
            createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
          }
          // vnode.text是否为原始值 也就是处理文本节点
        } else if (isPrimitive(vnode.text)) {
             // 利用createTextNode创建文本节点 并挂载到vnode.elm上
            nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
        }
    }
    
    function insert (parent, elm, ref) {
        // 定义了父元素
        if (isDef(parent)) {
            if (isDef(ref)) {
                // 判断ref的父节点和parent是否相同
                if (nodeOps.parentNode(ref) === parent) {
                    // 相同的话把elm(vnode对应的dom元素)插入ref之前
                    nodeOps.insertBefore(parent, elm, ref) 
                }
            } else {
                // 没有ref就append到parent中     
                nodeOps.appendChild(parent, elm)
            }
        }
    }
```

 ###### patchVnode
 ```javascript
    function patchVnode () {
    let oldCh = oldVnode.children
    let ch = vnode.children
        if (isDef(data) && isPatchable(vnode)) {
        // 调用sbs中的钩子函数，更新节点的属性/样式/事件
            for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
        // 用户的自定义钩子
        if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
        }
        
        
        if (isUndef(vnode.text)) {
            // 1. 新老节点都有子节点
            // 对子节点进行 diff操作调用updateChildren
            if (isDef(oldCh) && isDef(ch)) {
                if (oldCh !== ch) {
                     updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
                }
            // 2. 新节点有子节点，且老节点没有子节点
            } else if (isDef(ch)) {
                // 也就是老节点有文本节点
                if (isDef(oldVnode.text)) {
                    // 先清空老节点DOM的文本内容，然后当前DOM节点加入子节点
                    nodeOps.setTextContnt(elm, '')
                    // addVnodes把新节点下的子节点转换为DOM元素并且添加到dom树
                    addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
                }
            // 3. 老节点有子节点，且新节点没有子节点
            } else if (isDef(oldCh)) {
                // 删除老节点中的子节点 触发remove,destory钩子函数
                removeVnodes(oldCh, 0, oldCh.length - 1)
            
            // 4.  老节点有文本节点，新节点没有子节点
            } else if (isDef(oldVnode.text)) {
                // 清空老节点的文本内容
                nodeOps.setTextContent(elm, '')
            }
              
             // 新老节点都有文本节点，且文本不同，修改文本
        } else if (oldVnode.text !== vnode.text) {
            nodeOps.setTextContent(elm, vnode.text)
        }
        
        if (isDef(data)) {
            // 执行hook中postpatch  patch过程执行完毕
            if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
        }
    }
 ```
 
  ###### updateChildren  新旧节点都有子节点，且是sameVnode，对比新旧节点差异，更新到DOM树
  ```javascript
    function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
        let oldStartIdx = 0 // 旧开始索引
        let newStartIdx = 0 // 新开始索引
        let oldEndIdx = oldCh.length - 1 // 旧结束索引
        let oldStartVnode = oldCh[0] // 旧开始Vnode
        let oldEndVnode = oldCh(oldEndIdx) // 旧结束Vnode
        let newEndIdx = newCh.lengt - 1 // 新开始索引
        let newStartVnode = newCh[0] // 新开始Vnode
        let newEndVnode = newCh[newEndIdx] // 新结束Vnode
        let oldKeyToIdx, idxInOld, vnodeToMove, refElm
        
        // diff算法
        // 当新节点和旧节点都没有遍历完成
        while(oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
            // 1. 判断旧开始节点是否有值
            if (isUndef(oldStartVnode)) {
                // 不存在就将旧开始索引++ 获取后一个
                oldStartVnode = oldCh[++oldStartIdx]
            // 2. 判断旧结束节点是否有值
            } else if (isUndef(oldEndVnode)) {
                // 不存在就将旧结束索引-- 获取前一个
                oldEndVnode = oldCh[--oldEndIdx]
                
            // 3. 旧开始节点和新开始节点 是sameVnode
            } else if (sameVnode(oldStartVnode, newStartVnode)) {
                // sameVnode只说明tag key相同，所以调用patchVnode继续对比两节点，且将新开始节点，就开始节点索引及Vnode++
                patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
                oldStartVnode = oldCh[++oldStartIdx]
                newStartVnode = newCh[++newStartIdx]
             // 4. 旧结束节点和新结束节点 是sameVnode
            } else if (sameVnode(oldEndVnode, newEndVnode)) {
                patchVnode(oldEndVnod, newEndVnode, insertedVnodeQueue, newCh, newStartIdx)
                 oldEndVnode = oldCh[--oldEndIdx]
                 newEndVnode = newCh[--newEndIdx]
             // 5. 旧开始节点和新结束节点 是sameVnode
            } else if (sameVnode(oldStartVnode, newEndVnode)) {
                patchVnode(oldEndVnod, newEndVnode, insertedVnodeQueue, newCh, newStartIdx)
                oldStartVnode = oldCh[++oldStartIdx]
                newEndVnode = newCh[--newEndIdx]
                // 把旧的开始节点移动到旧的结束节点之后
                canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
            // 6. 旧结束节点和新开始节点 是sameVnode
            } else if (sameVnode(oldEndVnode, newStartVnode)) {
                patchVnode(oldEndVnod, newEndVnode, insertedVnodeQueue, newCh, newStartIdx)
                oldEndVnode = oldCh[--oldEndIdx]
                newStartVnode = newCh[++newStartIdx]
                canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
            } else {
                // 以上四种情况都不满足
                // 拿到新开始节点的key去旧节点数组中找相同key的旧节点
                
                // 先把老节点key和索引存在oldKeyToIdx，且没有赋值时，
                // 通过createKeyToOldIdx找出旧节点的key和对应的索引放入createKeyToOldIdx
                if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
                // 如果新开始节点有key， 用新开始节点的key去oldKeyToIdx数组中，找到对应旧节点对应key的索引
                // 如果新开始节点没有key，就去老节点数组中依次遍历找到相同老节点对应的索引
                idxInOld = isDef(newStartVnode.key)? oldKeyToIdx[newStartVnode.key] : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
          
                // 如果没有找到新开始节点对应老节点的索引
                if (isUndef(idxInOld)) { 
                     // 调用createElm创建新开始节点对应的dom对象，并插入老的开始节点对应DOM元素的前面
                     createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
                // 如果找到了
                } else {
                        // 取出对应的老节点
                        vnodeToMove = oldCh[idxInOld]
                        // 找到的对应节点与新开始节点是sameVnode
                        if (sameVnode(vnodeToMove, newStartVnode)) {
                             patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
                             oldCh[idxInOld] = undefined
                             canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
                        } else {
                                // 如果key相同但是是不同的元素(tag不同)，那么久创建新元素 
                                createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
                              }
                      }    
                       // 把下一个新开始节点 向后推一个
                       newStartVnode = newCh[++newStartIdx]
               }
        }
          // 当结束时 oldStartIdx  > oldEndIdx 旧节点遍历完，但是新节点还没有遍历完
          if (oldStartIdx > oldEndIdx) {
             // 说明新节点比老节点多，把剩下的新节点插入到老节点的后面
             refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
             addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
          // 当结束时 老节点没有遍历完 说明老节点多
          } else if (newStartIdx > newEndIdx) {
             // 把老节点数组中剩余节点批量删除
             removeVnodes(oldCh, oldStartIdx, oldEndIdx)
          }
        
    }
  ```
   
   ###### 模板编译
   
   1.vue2.x使用VNode描述视图以及各种交互，用户自己编写VNode比较复杂
   2.用户只需要编写类似于HTML的代码-Vue.js模板，通过编译器将模板转换为返回VNode的render函数
   3.vue文件会被webpack在构建的过程中转换成render函数(vue-loader)
   
   ```javascript    
    // src/core/instance/render.js
    //生产VNode
    _c() vm.createElement  
    // src/core/instance/render-helpers/index.js
    _m() renderStatic 
    // 创建文本VNode虚拟节点     
    _v() craeteTextVNode 
    // 转换字符串
    _s() toString
   
   ```
   
   ###### 编译模板的入口文件
   compileToFunctions
```javascript
   function createCompileToFunctionFn (compile){
        // 通过闭包缓存编译结果
        const cache = Object.create(null
        return function compileToFunctions (template, options, vm) {
            options = extend({}, options)
            const warn = options.warn || baseWarn
            delete options.warn
            
              // 1.读取缓存中的ComiledFunctionResult对象，如果有直接返回    
              // options.delimiters完整版vue有，改变插值表达式的符号 默认{{ }}
              const key = options.delimiters ? String(options.delimiters) +   template : template
              if (cache[key]) {
                 return cache[key]
              }
             
              // 2.把模板编译作为编译对象(render, staticRenderFns), 字符串形式的js代码
               const compiled = compile(template, options)
              // 3.把字符串形式的js代码转换成js方法(函数形式)
              const res = {}
              const fnGenErrors = []
              res.render = createFunction(compiled.render, fnGenErrors)
               res.staticRenderFns = compiled.staticRenderFns.map(code => {
                  return createFunction(code, fnGenErrors)
                })

               // 4.缓存并返回res对象(render, staticRenderFns方法)
               return (cache[key] = res)
   
        }
    }
```

##### compile函数 核心：合并选项，调用baseCompile进项编译，存储错误信息，返回编译好的对象 
```javascript
    function createCompiler (baseOptions) {
        // 参数： 1.模板   2.用户选项
        function compile (template, options) {
            // 原型指向baseOptions，作用用来合并baseOptions和compile函数传过来的compiler函数传过来的options
            const finalOptions = object.create(baseOptions)
            
            // 如果options存在
            if (options) {
                // 那么就合并options与baseOptions
            }
            
            
            // 调用baseCompile 模板编译核心函数
            // baseCompile 返回的是一个对象，两个成员分别是render函数和staticRendersFunctions
            // 此时的render存储的是字符串形式的js代码
            
            // baseCompile见下面代码块
            const compiled = baseCompile(template.trim(), finalOptions)
            
            return compiled
        } 
    }
```

##### 抽象语法树
抽象语法树简称AST
使用对象的形式描述树形的代码结构
此处的抽象语法树是用来描述树形结构的HTML字符串

为什么要使用抽象语法树
1. 模板字符串转换成AST后，可以通过AST对模板做优化处理
2. 标记模板中的静态内容，在patch的时候直接跳过静态内容(纯文本)
3. 在patch的过程中静态内容不需要对比

vue2.6  AST
![image.png](https://note.youdao.com/yws/res/12377/WEBRESOURCE7d49a1fe6ab43b6ead8ff4c91b48d90d)

```javascript
const createCompiler = createCompilerCreator (function baseCompile ( template, options) {
    // 把模板转换成ast抽象语法树
    // 抽象语法树，用来以树形的方式描述代码结构
    const ast = parse(template.trim(), otpions)
    if (options.optimize !== false) {
        // 优化抽象语法书
        optimize(ast, options)
    }
    
    // 把抽象语法树生成字符串形式的js代码
    const code = generate(ast, options)
    return {
        ast,
        // 渲染函数
        render: code.render,
        // 讲台渲染函数， 生成静态VNode树
        staticRenderFns: code.staticRenderFns
    }
    
})
   
```

###### baseCompile-parse 
parse函数内部依次遍历模板字符串，把模板字符串转换为AST对象，属性和指令都会记录在AST对象相应属性上
```javascript
// src/compiler/parser/index.js

// 参数1 模板字符串，合并后的选项
const ast = parse(template.trim(), options)

function parse() {
    // 1.解析options
    
    // 2.对模板解析
    parseHTML(template, {
        //解析指令
        // 解析过程中的回调函数， 生产AST
        start(tag, attrs, unary, start, end),
        end(tag, start, end),
        chars(text, start, end),
        comment(text, start)
    })
    
    // root存储的就是解析好的AST对象
    return root
}

```

###### baseCompile-optimize
```javascript
    // 优化抽象语法树
    optimize(art, options)
    
```

```javascript
    // 标记静态子树(文本标签) path时候直接使用
    function optimize(root, options) {
        // 是否传递了root (ast对象)
        // 如果没有直接返回，没有优化的必要
        if (!root) return
        // 标记静态节点
        markStatic(root)
        
        // 标记静态根节点
        markStaticRoots(root, false)
    }
    
    
    function markStatic (ASTNode) {
        // 判断当前astNode是否是静态的
        node.static = isStatic(node)
        // 元素节点
        if (node.type === 1) {
            // 不是保留标签，那就是组件
            // 不去把slot标记为静态节点，否则将来就不能改变
            if (!isPlatformReservedTag(node.tag) &&
                node.tag !== 'slot' &&
                node.attrsMap['inline-template'] == null）{
                    return
                }
            // 遍历子节点children
            for (let i = 0, l = node.children.length; i < l; i++) {
                 const child = node.children[i]
                 // 递归调用
                 markStatic(child)
                 if (!child.static) {
                     node.static = false
                    }
                }
            // 处理条件渲染中的AST对象
              if (node.ifConditions) {
                 for (let i = 1, l = node.ifConditions.length; i < l; i++) {
                     const block = node.ifConditions[i].block
                     markStatic(block)
                     if (!block.static) {
                       node.static = false
                     }
                 } 
            }
        }
    }
    
    function isStatic () {
        // 插值表达式
        if (node.type === 2) {
            return false
        }
        // 静态文本内容
        if (node.type === 3) {
            return true
        }
        
        // 都满足的话 是一个静态节点 返回true
         return !!(node.pre || ( // pre
            !node.hasBindings && // 没有动态绑定
            !node.if && // 不知指令
            !node.for && 
            !isBuiltInTag(node.tag) && // 不是内置组件
            isPlatformReservedTag(node.tag) &&  // 不是组件
            !isDirectChildOfTemplateFor(node)&& // 不是v-for下的直接子节点
            Object.keys(node).every(isStaticKey)
         ))
    }
    // 静态根节点： 标签中包含子标签，并且没有动态内容(纯文本内容)，如果标签中只包含纯文本内容，没有子标签，vue不会对其优化，因为优化成本大于收益
    function markStaticRoots (node: ASTNode, isInFor: boolean) {
        // ast描述的是否为元素
        if (node.type === 1) {
            if (node.static || node.once) {
                node.staticInFor = isInFor
            }
            // 如果一个元素内只有文本节点，此时这个元素不是静态根节点
            // 这种情况优化成本大于收益
            
            // 节点是静态的 && 有子节点 && 不能只有一个文本类型子节点
            if (node.static && node.children.length && !(
                node.children.length === 1 &&
                node.children[0].type === 3
             )) {
                 node.staticRoot = true
                 return
             } else {
               node.staticRoot = false
             }
             // 递归调用子节点
            if (node.children) {
               for (let i = 0, l = node.children.length; i < l; i++) {
                    markStaticRoots(node.children[i], isInFor || !!node.for)
                }
            }
            // 递归调用条件渲染
            if (node.ifConditions) {
               for (let i = 1, l = node.ifConditions.length; i < l; i++) {
                   markStaticRoots(node.ifConditions[i].block, isInFor)
               }
            }
        }
    }
```
###### baseCompile-generate 模板编译
```javascript
    // 把抽象语法树生成的字符串形式的js代码
    const code = generate(ast, options)
    
    
    function generate(ast, options) {
        const state = new CodegenState(options)
        const code = ast ? genElement(ast, state) : '_c("div")'
        
        return {
            render: `with(this){return${code}}`,
            staticRenderFns: state.staticRenderFns
        }
    }
    
    function genElement (el, state) {
        // 判断ast对象是否有parent属性
        if (el.parent) {
            // 记录pre属性(v-pre标记的都为静态节点)
            el.pre  = el.pre || el.parent.pre
        }
        
        // 如果已经被处理了，就不再处理（staticProcessed标记当前节点是否被处理）
        if (el.staticRoot && !el.staticProcessed) {
            return genStatic(el, state)
        } else if (el.once && !el.onceProcessed) {
            return genOnce(el, state)
        } else if (el.for && !el.forProcessed) {
            return genFor(el, state)
        } else if (el.if && !el.ifProcessed) {
            return genIf(el, state)
        } else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
            return genChilren(el, state) || 'void 0'
        } else if (el.tag === 'slot') {
            return genSlot(el, state)
        } else {
            let code 
            if (el.component) {
                code = genComponent(el.component, el, state)
            } else {
                let data
                if (!el.plain || (el.pre && state.maybeComponent(el))) {
                    // 将AST对象相应属性转换成createElement所需要的data对象的字符串形式
                    data = genData(el, state)
                }
                // 将AST对象子节点转换成createELement所需要的数组形式，也就是第三个参数
                const children = el.inlineTemplate ? null : genChildren(el, state, true)
                
                code = `_c('${el.tag}'${
                        data ? `,${data}` : '' // data
                         }${
                        children ? `,${children}` : '' // children
                        })`
                        
                for(let i = 0; i < state.transforms.length; i++) {
                    code = state.transforms[i](el, code)
                }
                return code
            }
        }
    }
    
    
    
    
function genStatic(el, state) {
    // 标记已经被处理
    el.staticProcessed = true
    const originalPreState = state.pre
    if (el.pre) {
        state.pre = el.pre
    }
    // 此时调用了genElement，但staticProcessed被标记为true，所以直接跳入该函数的else
    state.staticRenderFns.push(`with(this){return ${genElement(el, state)}}`)
    state.pre = originalPreState
    
    return `_m(${state.staticRenderFns.length - 1}${el.staticInFor ? ',true' : ''))`
}    
```
```

模板编译过程
1.模板字符串转换为AST对象
2.优化AST对象 标记静态根节点
3.优化好的AST对象转换为字符串形式的代码
4.将字符串形式的代码通过new Function转换为匿名函数
5.这个匿名函数就是最后的render函数


###### Vue.extend
```javascript
    // 定义一个唯一值cid的目的是：保证创建包裹的一个子构造函数通过原型继承并且能够缓存他们
    Vue.cid = 0;
    let cid = 1;
    Vue.extend = function (extendOptions) {
        extendOptions = extendOptions || {}
        
        const Super = this
        const Super = Super.cid
        // 从缓存中加载组件的构造函数，没有的话设置空对象
        const cacheCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
        // 通过cid获取缓存的构造函数
        if (cacheCtors[SuperId]) {
            return cachedCtors[SuperId]
        }
        
        const name = extendOptions.name || Super.options.name
        
        const Sub = function VueComponent(options) {
            // 调用_init()初始化
            this._init(options)
        }
        
        // 原型继承自Vue
        Sub.prototype = Object.create(Super.prototype)
        Sub.prototype.constructor = Sub
        Sub.cid = cid++
        // 合并options
        Sub.options = mergeOptions(
            Super.options,
            extendOptions 
        )
        
        Sub['super'] = Super
        
        if (Sub.options.props) {
            initProps(Sub)
        }
        
        if (Sub.options.computed) {
            initComputed(Sub)
        }
         // 继承静态方法
         Sub.extend = Super.extend
         Sub.mixin = Super.mixin
         Sub.use = Super.use
         
         ASSET_TYPES.forEach(function (type) {
             Sub[type] = Super[type]
         })
         // 在组件components中记录构造函数 
         if (name) {
            Sub.options.components[name] = Sub
         }
         
           Sub.superOptions = Super.options
           Sub.extendOptions = extendOptions
           Sub.sealedOptions = extend({}, Sub.options)

            // 把组件的构造函数缓存到options._Ctor
            cachedCtors[SuperId] = Sub
            // 返回sub构造函数
            return Sub
    }
```

###### 全局组件调试过程
```javascript
const Comp = Vue.component('comp', {
    template: '<div>Hello Component</div>'
})

const vm = new Vue({
    el: '#app',
    render (h) {
        return h(Comp)
    }
})

// initGlobalAPI中会调用该函数
// src/core/global-api/assets.js
function initAssetRegisters (Vue) {
    // ASSET_TYPE 包括directive component filter 
    ASSET_TYPE.forEach(type => {
        Vue[type] = function (id, definition) {
            // 如果没传第二个参数，则为调用
            if (!definition) {
                reurn this.options[type + 's'][id]
            } else {
                if (type === "component" && isPlainObject(definition)) {
                    definition.name = defaintion.name || id
                    // 调用extend方法
                    defintion = this.options._base.extend(defintion)
                    
                    // extend方法
                    // 1.初始化cid作为唯一标识
                    //2.从组件options中获取缓存的构造函数
                    // const xacheCtors = extendOptions._Ctor
                    // 3.创建组件构造函数
                    // const Sub = function VueComponent (options) { this._init(options)}
                    // 4. 原型继承自vue
                    //Sub.propto = Object.create(Super.prototype)
                    // Sub.prototype.constructor = Sub
                    // Sub.cid = cid++
                    // 5.合并sub的options(包括组件、指令、过滤器，_base(vue构造函数))和组件选项(name, template, _Ctor(用于缓存))
                    // Sub.option = mergeoptions(Subper.options, extendOptions)
                    // 6.初始化props，computed
                    // 7.继承vue静态成员 mixin use extend
                    // 8.缓存当前组件的构造函数cacheCtors[SuperId] = Sub, 此时extendOptions中_Ctor有了此组件构造函数
                    
                    // 9.跳出extend，将创建好的构造函数记录到vue构造函数options选项的component中 this.options[type + 's'][id] = definition （id 为注册的componet name）
                    } 
            }
        }
    })
}


```

######组件创建过程
```javascript
// src/core/vdom/create-element.js
function createElement (
) {
    // 1.处理参数
    // 2.调用_createElement(context, tag, data)
}
```