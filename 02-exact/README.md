## 起个头

前面已经介绍过了，关于 `_` 在内部是一个什么样的情况，其实就是定义了一个名字叫做 `_` 的函数，函数本身就是对象呀，就在 `_` 上扩展了 100 多种方法。

接着上一篇文章的内容往下讲，第一个扩展的函数是 `each` 函数，这和数组的 `forEach` 函数很像，即使不是数组，是伪数组，也可以通过 call 的方式来解决循环遍历，`forEach` 接受三个参数，且没有返回值，不对原数组产生改变。来看看 `each` 函数：

```javascript
_.each = _.forEach = function(obj, iteratee, context) {
  iteratee = optimizeCb(iteratee, context);
  var i, length;
  if (isArrayLike(obj)) {
    for (i = 0, length = obj.length; i < length; i++) {
      iteratee(obj[i], i, obj);
    }
  } else {
    var keys = _.keys(obj);
    for (i = 0, length = keys.length; i < length; i++) {
      iteratee(obj[keys[i]], keys[i], obj);
    }
  }
  return obj;
};
```

`each` 函数接收三个参数，分别是 obj 执行体，回调函数和回调函数的上下文，回调函数会通过 optimizeCb 来优化，optimizeCb 没有传入第三个参数 argCount，表明默认是三个，但是如果上下文 context 为空的情况下，就直接返回 iteratee 函数。

`isArrayLike` 前面已经介绍过了，不同于数组的 `forEach` 方法，`_` 的 each 方法可以处理对象，只不过要先调用 `_.keys` 方法获取对象的 keys 集合。返回值也算是一个特点吧，each 函数返回 obj，而数组的方法，是没有返回值的。

第二个是 `map` 函数：

```javascript
_.map = _.collect = function(obj, iteratee, context) {
  iteratee = cb(iteratee, context); // 回调函数
  var keys = !isArrayLike(obj) && _.keys(obj), // 处理非数组
      length = (keys || obj).length,
      results = Array(length);
  for (var index = 0; index < length; index++) {
    var currentKey = keys ? keys[index] : index; // 数组或者对象
    results[index] = iteratee(obj[currentKey], currentKey, obj);
  }
  return results;
};
```

套路都是一样的，既可以处理数组，又可以处理对象，但是 map 函数要有一个返回值，无论是数组，还是对象，返回值是一个数组，而且从代码可以看到，新生成了数组，不会对原数组产生影响。

然后就是 `reduce` 函数，感觉介绍完这三个就可以召唤神龙了，其中 reduce 分为左和右，如下：

```javascript
_.reduce = _.foldl = _.inject = createReduce(1);

_.reduceRight = _.foldr = createReduce(-1);
```

为了减少代码量，就用 `createReduce` 函数，接收 1 和 -1 参数：

```javascript
function createReduce(dir) {
  // iterator 函数是执行，在最终结果里面
  function iterator(obj, iteratee, memo, keys, index, length) {
    for (; index >= 0 && index < length; index += dir) {
      var currentKey = keys ? keys[index] : index;
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }
    return memo;
  }

  return function(obj, iteratee, memo, context) {
    // 依旧是回调函数，优化
    iteratee = optimizeCb(iteratee, context, 4);
    var keys = !isArrayLike(obj) && _.keys(obj),
        length = (keys || obj).length,
        index = dir > 0 ? 0 : length - 1;
    // 参数可以忽略第一个值，但用数组的第一个元素替代
    if (arguments.length < 3) {
      memo = obj[keys ? keys[index] : index];
      index += dir;
    }
    return iterator(obj, iteratee, memo, keys, index, length);
  };
}
```

`createReduce` 用闭包返回了一个函数，该函数接受四个参数，分别是执行数组或对象、回调函数、初始值和上下文，个人感觉这里的逻辑有点换混乱，比如我只有三个参数，有初始值没有上下文，这个好办，但是如果同样是三个参数，我是有上下文，但是没有初始值，就会导致流程出现问题。不过我也没有比较好的解决办法。

当参数为两个的时候，初始值没有，就会调用数组或对象的第一个参数作为初始值，并把指针后移一位（这里用指针，其实是数组的索引），传入的函数，它有四个参数，这和数组 reduce 方法是一样的。

个人认为，reduce 用来处理对象，还是有点问题的，比如获取对象的 keys 值，如果每次获取的顺序都不一样，导致处理的顺序也不一样，那最终的结果还会一样吗？

