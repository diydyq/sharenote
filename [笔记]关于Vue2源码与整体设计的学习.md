# [笔记]关于Vue2源码与整体设计的学习


## Vue2主要分为几个部分

Vue2是在16年10月推出，优势较之前很明显，所以团队里升级很快，并且围绕Vue2源码学习做一个分享，从数据驱动框架的角度上整体分为5个模块：

1. Setter/Getter代理：UI界面层对数据的读写
2. Observe类、Dep类、Watcher类：完成Component组件与Expression表达式（如：{{ ... }}}）的依赖管理
3. 模板编译前置AOT（Ahead Of Time）：将组件模板编译为DOM树节点，每个节点以函数的形式体现
4. VNode的渲染：VNode与Document Element的转换
5. Virtual-DOM中新旧VNode的对比：两颗VNode树节点，如何以最优的算法，找到不同点并进行更新


## 模块1：Setter/Getter代理

众所周知，Vue1&2里都利用了JS的Getter/Setter完成UI层中数据的读写，那不可避免的就必然会用到一个API：`Object.defineProperty(obj, key, { ... })`，源码中的使用有以下几处：

```javascript
// 1. vueInstance.initData()对data属性中的每条数据key做代理，将每条key定义在组件实例上
// 注意：这个调用仅涉及data的直接属性，深层次的setter/getter是另一个方法
function proxy (vm, key) {
  if (!isReserved(key)) {
    Object.defineProperty(vm, key, {
      configurable: true,
      enumerable: true,
      get: function proxyGetter () {
        return vm._data[key]
      },
      set: function proxySetter (val) {
        vm._data[key] = val;
      }
    });
  }
}

function initData (vm) {
  var data = vm.$options.data;
  data = vm._data = typeof data === 'function'
    ? data.call(vm)
    : data || {};
  if (!isPlainObject(data)) {
    data = {};
    "development" !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    );
  }
  // proxy data on instance
  var keys = Object.keys(data);
  var props = vm.$options.props;
  var i = keys.length;
  while (i--) {
    if (props && hasOwn(props, keys[i])) {
      "development" !== 'production' && warn(
        "The data property \"" + (keys[i]) + "\" is already declared as a prop. " +
        "Use prop default value instead.",
        vm
      );
    } else {
      proxy(vm, keys[i]);
    }
  }
  // observe data
  observe(data, true /* asRootData */);
}
```

```javascript
// 2. vueInstance.initComputed()对computed属性中的每条数据做代理，这里方便直接定义Getter，所以Setter为noop空函数；
function initComputed (vm, computed) {
  for (var key in computed) {
    /* istanbul ignore if */
    if ("development" !== 'production' && key in vm) {
      warn(
        "existing instance property \"" + key + "\" will be " +
        "overwritten by a computed property with the same name.",
        vm
      );
    }
    var userDef = computed[key];
    if (typeof userDef === 'function') {
      computedSharedDefinition.get = makeComputedGetter(userDef, vm);
      computedSharedDefinition.set = noop;
    } else {
      computedSharedDefinition.get = userDef.get
        ? userDef.cache !== false
          ? makeComputedGetter(userDef.get, vm)
          : bind$1(userDef.get, vm)
        : noop;
      computedSharedDefinition.set = userDef.set
        ? bind$1(userDef.set, vm)
        : noop;
    }
    Object.defineProperty(vm, key, computedSharedDefinition);
  }
}
```

```javascript
// 3. 对属性值为obj字典对象的属性代理
function defineReactive$$1 (
  obj,
  key,
  val,
  customSetter
) {
  var dep = new Dep();

  var property = Object.getOwnPropertyDescriptor(obj, key);
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  var getter = property && property.get;
  var setter = property && property.set;

  var childOb = observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val;
      if (Dep.target) {
        dep.depend();
        if (childOb) {
          childOb.dep.depend();
        }
        if (Array.isArray(value)) {
          dependArray(value);
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if ("development" !== 'production' && customSetter) {
        customSetter();
      }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = observe(newVal);
      dep.notify();
    }
  });
}
```

```javascript
// 4. 公用Util
// 4.1 如：代理数组原型方法（'push','pop','shift','unshift','splice','sort','reverse'），在数组实例修改时触发脏数据检查
// 4.2 如：为data中的对象定义key为__ob__的观察对象
function def (obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  });
}
```

```javascript
// 5. 避免直接对vueInstance.$data和Vue.config的直接直接：
// vueInstance.$data
Object.defineProperty(Vue.prototype, '$data', dataDef);
// Vue.config
Object.defineProperty(Vue, 'config', configDef);
```


## 模块2：Observe类、Dep类、Watcher类

如上所述，UI层修改时肯定会调用setter方法，但是修改之后是如何做到更新依赖的呢？它的依赖包括组件还是依赖该属性的其它表达式呢？这里的问题主要有3点：

1. 依赖于属性的组件/表达式怎么收集的？
2. 依赖的组件/表达式接下来怎么更新？
3. 依赖管理是什么样子？好理解吗？

从上面`代码块：3. 对属性值为obj字典对象的属性代理`的代码中，我们可以看到：对象`obj`中的每个属性`key`原本的值`val`都会重新以`defineProperty()`的方式重新定义；同时针对每个属性`key`，都会以闭包的形式定义对应的`Dep`实例`dep`，看来属性`key`与实例`dep`是一一对应的关系，那么使用`dep`在`getter`时收集依赖方，`setter`时通知依赖方是不是一种很好的方式呢？

确实！Vue2就是这么做的，`dep.depend();`负责收集依赖，`dep.notify();`负责通知依赖，如下面的源码片段：

```javascript
// 收集依赖
var Dep = function Dep () {
  this.id = uid$1++;
  this.subs = [];
};

Dep.prototype.depend = function depend () {
  if (Dep.target) {
    Dep.target.addDep(this);
  }
};

Watcher.prototype.addDep = function addDep (dep) {
  var id = dep.id;
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id);
    this.newDeps.push(dep);
    if (!this.depIds.has(id)) {
      // 将依赖方Watcher加入dep.subs数组中
      dep.addSub(this);
    }
  }
};
```

```javascript
// 通知依赖
Dep.prototype.notify = function notify () {
  var subs = this.subs.slice();
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update();
  }
};

Watcher.prototype.update = function update () {
  if (this.lazy) {
    this.dirty = true;
  } else if (this.sync) {
    this.run();
  } else {
    queueWatcher(this);
  }
};
```

根据代码`dep.addSub(this);`看来依赖方一定是个`Watcher`；那`Watcher`代表的是啥？看看构造函数与调用场景才能得知：


```javascript
// 传递vue组件，deps,depIds记录组件中的key调用
var Watcher = function Watcher (
  vm,
  expOrFn,
  cb,
  options
) {
  this.vm = vm;
  vm._watchers.push(this);
  // options
  if (options) {
    this.deep = !!options.deep;
    this.user = !!options.user;
    this.lazy = !!options.lazy;
    this.sync = !!options.sync;
  } else {
    this.deep = this.user = this.lazy = this.sync = false;
  }
  this.cb = cb;
  this.id = ++uid$2; // uid for batching
  this.active = true;
  this.dirty = this.lazy; // for lazy watchers
  this.deps = [];
  this.newDeps = [];
  this.depIds = new _Set();
  this.newDepIds = new _Set();
  this.expression = expOrFn.toString();
  // parse expression for getter
  if (typeof expOrFn === 'function') {
    this.getter = expOrFn;
  } else {
    this.getter = parsePath(expOrFn);
    if (!this.getter) {
      this.getter = function () {};
      "development" !== 'production' && warn(
        "Failed watching path: \"" + expOrFn + "\" " +
        'Watcher only accepts simple dot-delimited paths. ' +
        'For full control, use a function instead.',
        vm
      );
    }
  }
  this.value = this.lazy
    ? undefined
    : this.get();
};

// 调用1：在组件挂载mount的生命周期中实例化
Vue.prototype._mount = function (
  el,
  hydrating
) {
  var vm = this;
  vm.$el = el;
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode;
  }
  callHook(vm, 'beforeMount');
  vm._watcher = new Watcher(vm, function () {
    vm._update(vm._render(), hydrating);
  }, noop);
  hydrating = false;
  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true;
    callHook(vm, 'mounted');
  }
  return vm
};
// 调用2：代理computed属性的getter自定义方法，dirty后重新执行
function makeComputedGetter (getter, owner) {
  var watcher = new Watcher(owner, getter, noop, {
    lazy: true
  });
  return function computedGetter () {
    if (watcher.dirty) {
      watcher.evaluate();
    }
    if (Dep.target) {
      watcher.depend();
    }
    return watcher.value
  }
}
// 调用3：在watch属性中的使用
Vue.prototype.$watch = function (
  expOrFn,
  cb,
  options
) {
  var vm = this;
  options = options || {};
  options.user = true;
  var watcher = new Watcher(vm, expOrFn, cb, options);
  if (options.immediate) {
    cb.call(vm, watcher.value);
  }
  return function unwatchFn () {
    watcher.teardown();
  }
};

```

可以看出，`Watcher`就是一个监听器，属性`deps, depIds`记录了每一个要监听的对象，当它们发生变化时，触发监听器的更新。那么更新的内容都包括哪些呢？很明显就是调用`new Watcher()`的地方了，即向构造函数传递`expOrFn`的参数；代码中显示了3处：

1. 当Vue组件渲染更新时，包括首次挂载时，随后模板`render`更新
2. 当computed中某个属性key的`getter`函数声明中的某个变量更新时，触发该`getter`函数的重新执行
3. 同理`watch`属性；



## 是否有继续优化的空间








