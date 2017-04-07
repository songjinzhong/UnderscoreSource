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
  _.each(_.functions(obj), function(name) {
    var func = _[name] = obj[name];
    _.prototype[name] = function() {
      var args = [this._wrapped];
      push.apply(args, arguments);
      return result(this, func.apply(_, args));
    };
  });
};

_.mixin(_); // 执行

_.functions = _.methods = function(obj) {
  var names = [];
  for (var key in obj) {
    if (_.isFunction(obj[key])) names.push(key);
  }
  return names.sort();
};
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
  instance._chain = true;
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

现在返回来看一下 `_` 函数：

```javascript
var _ = function(obj) {
  if (obj instanceof _) return obj; // 如果 obj 继承于 _，直接返回
  if (!(this instanceof _)) return new _(obj); // 如果 this 不继承 _，返回一个 new
  this._wrapped = obj; // 保存 obj 的值
};
```

现在应该就非常的明朗了吧。