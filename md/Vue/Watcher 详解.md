本篇文章将会记录本人对 Vue 中 Watcher 的理解，版本为 Vue 2.7.3，适合对 Vue 如何进行观察者和订阅/发布已有一定了解的人。

首先先抛出几个问题，后文将会一一解答
- Watcher 是什么？他的作用？
- Watcher 有多少种类型？如何区分？
- 不同类型的 Watcher 做了什么不同的处理？

<details>
<summary> Watcher 类代码</summary>

``` typescript
// src\core\observer\watcher.ts

// export interface DebuggerOptions {
//   onTrack?: (event: DebuggerEvent) => void
//   onTrigger?: (event: DebuggerEvent) => void
// }

// export interface DepTarget extends DebuggerOptions {
//   id: number
//   addDep(dep: Dep): void
//   update(): void
// }

export default class Watcher implements DepTarget {
  vm?: Component | null
  expression: string
  cb: Function
  id: number
  deep: boolean
  user: boolean
  lazy: boolean
  sync: boolean
  dirty: boolean
  active: boolean
  deps: Array<Dep>
  newDeps: Array<Dep>
  depIds: SimpleSet
  newDepIds: SimpleSet
  before?: Function
  onStop?: Function
  noRecurse?: boolean
  getter: Function
  value: any

  // dev only
  onTrack?: ((event: DebuggerEvent) => void) | undefined
  onTrigger?: ((event: DebuggerEvent) => void) | undefined

  constructor(
    vm: Component | null,
    expOrFn: string | (() => any),
    cb: Function,
    options?: WatcherOptions | null,
    isRenderWatcher?: boolean
  ) {
    recordEffectScope(this, activeEffectScope || (vm ? vm._scope : undefined))
    if ((this.vm = vm)) {
      if (isRenderWatcher) {
        vm._watcher = this
      }
    }
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
      if (__DEV__) {
        this.onTrack = options.onTrack
        this.onTrigger = options.onTrigger
      }
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = __DEV__ ? expOrFn.toString() : ''
    // parse expression for getter
    if (isFunction(expOrFn)) {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        __DEV__ &&
          warn(
            `Failed watching path: "${expOrFn}" ` +
              'Watcher only accepts simple dot-delimited paths. ' +
              'For full control, use a function instead.',
            vm
          )
      }
    }
    this.value = this.lazy ? undefined : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get() {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e: any) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep(dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps() {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp: any = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update() {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run() {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          const info = `callback for watcher "${this.expression}"`
          invokeWithErrorHandling(
            this.cb,
            this.vm,
            [value, oldValue],
            this.vm,
            info
          )
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate() {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend() {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown() {
    if (this.vm && !this.vm._isBeingDestroyed) {
      remove(this.vm._scope.effects, this)
    }
    if (this.active) {
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
      if (this.onStop) {
        this.onStop()
      }
    }
  }
}
```
</details>

## Watcher 是什么？他的作用？
在代码中，作者是如此注释的：
``` typescript
/**
 * A watcher parses an expression, collects dependencies,
 * and fires callback when the expression value changes.
 * This is used for both the $watch() api and directives.
 * @internal
 */

 /**一个观察者解析一个表达式，收集依赖，
    并在表达式值更改时触发回调。
    这用于 $watch() api 和指令。
 */
```
Watcher 即观察者，通过订阅 Dep ( ```Watcher.get()``` )后， Dep 便可以发布更新时通知 Watcher 调用对应回调 ( ```Watcher.get() => Watcher.run() => Watcher.cb.call(this.vm, value, oldValue)``` )。
Vue 中的作用是，订阅某个值的变化，此值变化时便进行视图的更新。

## Watcher 有多少种类型？如何区分？
关于 Watcher 的类型，我们可以从源码 /src 路径下全局搜索```new Watcher```开始分析。
这里用伪代码简单概括下：
``` typescript 
// 渲染 Watcher ，在 beforeMount 和 mounted 之间创建
// src\core\instance\lifecycle.ts
updateComponent = () => {
    vm._update(vm._render(), hydrating)
}
const watcherOptions: WatcherOptions = {
    before() {
        if (vm._isMounted && !vm._isDestroyed) {
            callHook(vm, 'beforeUpdate')
        }
    }
}

new Watcher(
    vm,
    updateComponent,
    noop, // function noop(a?: any, b?: any, c?: any) {}
    watcherOptions,
    true /* isRenderWatcher */
)

// computed Watcher ，在 beforeCreate 和 created 之间的 initState() => initComputed() 中创建
// vm.$options.computed 每个属性均对应一个 computed Watcher
// src\core\instance\state.ts
const computedWatcherOptions = { lazy: true }

const watchers = (vm._computedWatchers = Object.create(null))
watchers[key] = new Watcher(
    vm,
    getter || noop, // 用户创建的 get || noop
    noop,
    computedWatcherOptions
)

// user Watcher ，在 beforeCreate 和 created 之间的 initState() => initWatch() 中创建
// vm.$options.watch 上一个属性若是数组，则数组的每个 key 均对应一个 user Watcher；若非数组，则本身对应一个 user watcher
// src\core\instance\state.ts

interface ComponentOptions {
  watch?: {
    [key: string]: WatchOptionItem | WatchOptionItem[]
  }
}

type WatchOptionItem = string | WatchCallback | ObjectWatchOptionItem

type WatchCallback<T> = (
  value: T,
  oldValue: T,
  onCleanup: (cleanupFn: () => void) => void
) => void

type ObjectWatchOptionItem = {
  handler: WatchCallback | string
  immediate?: boolean // default: false
  deep?: boolean // default: false
  flush?: 'pre' | 'post' | 'sync' // default: 'pre'
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
}

Vue.prototype.$watch = function (
    expOrFn: string | (() => any),
    cb: any,
    options?: Record<string, any>
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      const info = `callback for immediate watcher "${watcher.expression}"`
      pushTarget()
      invokeWithErrorHandling(cb, vm, [watcher.value], vm, info)
      popTarget()
    }
    return function unwatchFn() {
      watcher.teardown()
    }
}

// TIPS：这里可以看出 watch 支持异步而 computed 不支持异步的原因，因为 User Watcher 创建时有传用户配置的 cb 参数而创建 Computed Watcher 时传的是 noop；而在 watch 中，能接受到( oldValue , newValue ) 的是 cb

```

总结一下， Watcher 有三种类型：
- Render Watcher ，即渲染 Watcher
  
  通过传入的最后一个参数即 isRenderWatcher 来区分，值为 true 时则为渲染 Watcher 

- Computed Watcher ，即 computed 属性生成的 watcher

  在第四个参数 options 对象中设置 lazy : true 来区分

- User Watcher ，即 watch 属性或者 $watch api 生成的 watcher
  
  在第四个参数 options 对象中设置 user : true 来区分

## 不同类型的 Watcher 做了什么不同的处理？


|  类型/阶段   | 构造阶段  | run()  |  update()  |  调用的回调  |
|  ----  | ----  | ----  | ----  | ----  |
| Render Watcher  | 将本 Watcher 实例赋值给 vm._watcher ，一个 VUe 实例对应一个渲染 Watcher | noop |
| Computed Watcher  | 设置 ```this.dirty = this.lazy = !!options.lazy```，并通过 ```this.lazy``` 来判断是否立即执行 watcher.get() 并赋值给 Watcher.value。若 lazy == true ，则不是立刻执行。 | / |  Watcher.update() 时把 this.dirty 设置为 true 后即返回，此行为与 computed 的缓存有关  | noop |
| User Watcher  | / | 若是user watcher```{invokeWithErrorHandling(...)}```，若不是```this.cb.call(this.vm, value, oldValue)```  | / | 用户创建的回调 |
