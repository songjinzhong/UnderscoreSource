# 总体架构

其实，学习一个库的源码，最重要的就是先理清它的基本架构，jQuery 是这样，Underscore 也应该是这样。

Underscore 这个库提供力很多有用的函数，这些函数部分已经在 es5 或 es6 中支持了，比如我们常用的 map、reduce、each，还有 es6 中的 keys 方法等，因为这些方法比较好用，所以被 javascript 的制定者采纳了。

## 先过一遍源码

我看的版本是 1.8.3，网上很多旧版本的，貌似有很多函数都已经启用或改变了，有点不一样啦。

打开源码，会看到函数的基本架构：

```javascript
(function(){
  ...
}.call(this))
```

这和我们常见的闭包不太一样啊，但是功能都是类似的，在函数内执行，防止对全局变量进行污染，然后在函数的最后调用 call 函数，把 函数内部的 this 和全局的 this 进行绑定。如果在浏览器里执行，this 会指向 window，在 node 环境下，会指向全局的 global。当厌倦使用闭包的时候，这种方法也是一种不错的体验。

那么接着向下看：

```javascript
(function(){
  var root = this; // 用 root 来保存当前的 this
  var previousUnderscore = root._; // 万一 _ 之前被占用了，先备份

  // 下面是一些原型，包括 数组，对象和函数
  var ArrayProto = Array.prototype, ObjProto = Object.prototype, FuncProto = Function.prototype;

  var
    push             = ArrayProto.push,
    slice            = ArrayProto.slice,
    toString         = ObjProto.toString,
    hasOwnProperty   = ObjProto.hasOwnProperty;

  var
    nativeIsArray      = Array.isArray,
    nativeKeys         = Object.keys,
    nativeBind         = FuncProto.bind,
    nativeCreate       = Object.create;
}.call(this))
```

在源码中搜索 `previousUnderscore`，可以找到两处，另外一处就是：

```javascript
_.noConflict = function() {
  root._ = previousUnderscore;
  return this;
};
```

`noConflict` 函数的用法是可以让用户自定义变量来替代 `_`，并且把之前保存的 `_` 给还原，比如：

```javascript
// 测试使用
var _ = 'Hello World'; // _ 已经被占用
...

var us = _.noConflict();
us // 指向 underscore
_ // ‘Hello World'
```

像 push slice 这些函数，原生都已经支持了，源码里面直接把他们拿过来使用。

## 来看看 _ 是如何定义的

`_` 在 underscore 地位是非常核心的，而它的本质实际上还是函数，同样也是一个对象：

```javascript
var _ = function(obj) {
  if (obj instanceof _) return obj;
  if (!(this instanceof _)) return new _(obj);
  this._wrapped = obj;
};
_.VERSION = '1.8.3';
_.map = _.collect = function(){...};
_.each = _.forEach = function(){...};
```

`_` 是一个函数，但是在源码中，它是被当作对象来使用，所有的属性和函数都是直接绑定到 `_` 对象上面的，所有最终的调用都是通过：

```javascript
_.each([22,33,44], console.log);
// 22 0 [22, 33, 44]
// 33 1 [22, 33, 44]
// 44 2 [22, 33, 44]
```

最终的返回值是处理的那个数组，而不是 `_` 自己，下面将会讨论，这个涉及到链式调用。

那如果，我就想通过函数来生成，这也是支持的：

```javascript
_([1,2,3]).each(console.log)
// 返回的结果都是一样的
```

这个时候，就会疑惑，`_` 的原型呢？我们再来搜索一下 `_.prototype`：

```javascript
_.mixin = function(obj) {
  _.each(_.functions(obj), function(name) { // 调用 each 对每一个函数对象处理
    var func = _[name] = obj[name]; // 绑定到 _ 上
    _.prototype[name] = function() { // 绑定到 _ 的原型上
      var args = [this._wrapped];
      push.apply(args, arguments); // 参数对齐
      return result(this, func.apply(_, args)); // 调用 result 查看是否链式
    };
  });
};

_.mixin(_); // 执行

// 相关的一些方法
_.functions = _.methods = function(obj) {
  var names = [];
  for (var key in obj) {
    if (_.isFunction(obj[key])) names.push(key);
  }
  return names.sort();
};
_.isFunction = function(obj){
  return typeof obj == 'function' || false;
}
```

`_.functions` 是一个获取目标所有函数对象的方法，并把这些方法浅拷贝传递给 `_` 和 `_`的原型，因为原型方法，处理对象已经在 `_wrapped` 中了，而这些常用的方法参数都是固定的，如果直接调用，参数会出问题，所以：

```javascript
var args = [this._wrapped];
push.apply(args, arguments);// args 已经拼接完成
func.apply(_, args);
```

那么 result 函数是用来做什么的？因为 underscore 有两种调用方式，一种是通过 `_.each(obj, func)`，另一种是通过 `_(obj).each(func)`。第一种方法很好理解，返回值要么是 obj 本身，要么是处理后的结果，而第二种调用方法和 jQuery 很像，先生成一个 new 实体，对实体的进行调用，也就有了上面的参数校准问题。

不过这样子又回带来另一个问题，对于 each、map 函数，函数返回什么不重要，主要是处理过程，可以支持链式调用，对于 reduce 函数，返回的是处理后的结果，可以不用链式，所以 result 函数就是来判断是否需要链式，而对返回值进行处理。

介绍 result 之前，先来看一下 chain 函数：

```javascript
_.chain = function(obj) {
  var instance = _(obj);
  instance._chain = true; // 设置一个 _chain 属性，后面用于判断链式
  return instance;
};
```

返回一个新的 `_(obj)`，并且多了一个 `_chain` 属性，且为 true，所以 result 函数：

```javascript
var result = function(instance, obj) {
  return instance._chain ? _(obj).chain() : obj;
};
```

如果当前是允许链式的，可以进行链式调用，不允许链式，就直接返回处理结果，比如：

```javascript
var arr = [22, 33, 44];
_.chain(arr)
  .map(function(v){ return v + 1 })
  .reduce(function(p, n){ return p + n }, 0)
  .value() // 102

// 如果不允许链式，返回结果是处理后的数组
_(arr)
  .map(function(v){ return v + 1 }) // [23, 34, 45]
```

现在返回来看一下 `_` 函数，也非常的有意思，`_(obj)`实际上是执行两次的，第二次才用到了 new：

```javascript
var _ = function(obj) {
  if (obj instanceof _) return obj; // 如果 obj 继承于 _，直接返回
  if (!(this instanceof _)) return new _(obj); // 如果 this 不继承 _，返回一个 new
  this._wrapped = obj; // 保存 obj 的值
};
```

现在应该就非常的明朗了吧。当调用 `_([22,33,44])` 的时候，发现 obj 并不是继承与 `_`，会用 new 来生成，又会重新跑一遍 `_` 函数，然后将 `_wrapped` 属性指向 obj。

由于在之前已经 `root = this`，Underscore 在不同的环境中都可以运行，需要将 `_` 放到不同的环境中：

```javascript
if (typeof exports !== 'undefined') { // nodejs 模块
  if (typeof module !== 'undefined' && module.exports) {
    exports = module.exports = _;
  }
  exports._ = _;
} else { // window
  root._ = _;
}
```

## 接着看源码

源码再往下看，是一个 optimizeCb 函数，用来优化回调函数：

```javascript
var optimizeCb = function(func, context, argCount) {
  // 这里没有用 undefined，而是用 void 0
  if (context === void 0) return func; // 只有一个参数，直接返回回调函数
  switch (argCount == null ? 3 : argCount) { // call 比 apply 好？
    case 1: return function(value) {
      return func.call(context, value);
    };
    case 2: return function(value, other) {
      return func.call(context, value, other);
    };
    case 3: return function(value, index, collection) {
      return func.call(context, value, index, collection);
    };
    case 4: return function(accumulator, value, index, collection) {
      return func.call(context, accumulator, value, index, collection);
    };
  }
  // 最后走 apply 函数
  return function() {
    return func.apply(context, arguments);
  };
};
```

所谓优化版的回调函数，就是用 call 来固定参数，1 个参数，2 个参数，3 个参数，4 个参数的时候，由于 apply 可以不用考虑参数，但是在性能上面貌似没有 call 好。

然后后面还有一个 cb 函数，也是用来作为回调函数的。

```javascript
var cb = function(value, context, argCount) {
  if (value == null) return _.identity;
  if (_.isFunction(value)) return optimizeCb(value, context, argCount);
  if (_.isObject(value)) return _.matcher(value);
  return _.property(value);
};
_.iteratee = function(value, context) {
  return cb(value, context, Infinity);
};
```

iteratee 可以用来对函数进行处理，给一个函数绑定 this 等等，最总还是调用到 cb，其实 cb 本身就很复杂，要么是一个 identity 函数，要么是一个优化到回调函数，要么是一个 property 获取属性函数。

再往下就是 `createAssigner`，搜了一下，发现全文有三处用到此函数，分别是 extend、extendOwn、default，可以看出来，此函数主要到作用是用来实现拷贝，算是拷贝到辅助函数吧，把拷贝公共到部分抽离出来：

```javascript
var createAssigner = function(keysFunc, undefinedOnly) {
  return function(obj) {
    var length = arguments.length;
    if (length < 2 || obj == null) return obj;

    // 将第二个参数及以后的 object 拷贝到第一个 obj 上
    for (var index = 1; index < length; index++) {
      var source = arguments[index],
          // keysFunc 是点睛所在
          // 不同的 keysFunc 获得的 keys 集合不同
          // 分为两种，所有 keys（包括继承），自身 keys
          keys = keysFunc(source),
          l = keys.length;
      for (var i = 0; i < l; i++) {
        var key = keys[i];
        // underfinedOnly 表示是否覆盖原有
        if (!undefinedOnly || obj[key] === void 0) obj[key] = source[key];
      }
    }
    return obj;
  };
};
```

所以当 keyFunc 函数获得所有 keys 时，包括继承来的，这个时候就对应于 `_.extend` 函数，非继承 keys 时，对应于 `_.extendOwn`。如果 underfinedOnly 设置为 true，则实现的是不替换原有属性的继承 `_.defaults`。

在 Underscore 中，原型的继承用 baseCreate 函数：

```javascript
var Ctor = function(){};

var baseCreate = function(prototype) {
  if (!_.isObject(prototype)) return {};
  if (nativeCreate) return nativeCreate(prototype);
  Ctor.prototype = prototype;
  var result = new Ctor;
  Ctor.prototype = null;
  return result;
};
```

`nativeCreate` 之前已经介绍来，就是 `Object.create`，所以，如果浏览器不支持，下面实现的功能就是在实现这个函数，方法也很常规，用了一个空函数 Ctor 主要是防止 new 带来的多余属性问题。

`property` 函数也是一个比较有意思的函数，使用了闭包的思路，比如判断一个对象是否为类似数组结构的时候就用到了这个函数：

```javascript
var property = function(key) {
  return function(obj) {
    return obj == null ? void 0 : obj[key];
  };
};

var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1;
var getLength = property('length'); // 返回一个闭包韩式，用来检测对象是非有 length 参数
var isArrayLike = function(collection) {
  var length = getLength(collection);
  return typeof length == 'number' && length >= 0 && length <= MAX_ARRAY_INDEX;
};
```

而且我搜索了一下，发现 getLength 函数使用的地方还是挺多的。

## 参考

>[Underscore.js (1.8.3) 中文文档](http://www.css88.com/doc/underscore/)

>[Underscore源码解析（一）](https://segmentfault.com/a/1190000000515420)

>[中文版 underscore 代码注释](https://github.com/hanzichi/underscore-analysis/blob/master/underscore-1.8.3.js/underscore-1.8.3-analysis.js)