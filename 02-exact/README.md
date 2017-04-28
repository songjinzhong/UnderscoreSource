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

个人认为，reduce 用来处理对象，还是有点问题的，比如获取对象的 keys 值，如果每次获取的顺序都不一样，导致处理的顺序也不一样，那最终的结果还会一样吗？所以我决定处理对象还是要谨慎点好。

```javascript
_.findKey = function(obj, predicate, context) {
  predicate = cb(predicate, context);
  var keys = _.keys(obj), key;
  for (var i = 0, length = keys.length; i < length; i++) {
    key = keys[i];
    if (predicate(obj[key], key, obj)) return key;
  }
};
_.find = _.detect = function(obj, predicate, context) {
  var key;
  if (isArrayLike(obj)) {
    key = _.findIndex(obj, predicate, context);
  } else {
    key = _.findKey(obj, predicate, context);
  }
  if (key !== void 0 && key !== -1) return obj[key];
};
```

`find` 也是一个已经在数组方法中实现的，对应于数组的 `find` 和 `findIndex` 函数。在 `_`中，`_.findKey` 针对于对象，`_.findIndex` 针对于数组，又略有不同，但是讨论和 reduce 的套路是一样的：

```javascript
function createPredicateIndexFinder(dir) {
  return function(array, predicate, context) {
    predicate = cb(predicate, context);
    var length = getLength(array);
    var index = dir > 0 ? 0 : length - 1;
    for (; index >= 0 && index < length; index += dir) {
      if (predicate(array[index], index, array)) return index;
    }
    return -1;
  };
}

_.findIndex = createPredicateIndexFinder(1);
_.findLastIndex = createPredicateIndexFinder(-1);
```

## 有重点的来看

后面觉得有的函数真的是太无聊了，套路都是一致的，仔细看了也学不到太多的东西，感觉还是有选择的来聊聊吧。[underscore-analysis](https://github.com/hanzichi/underscore-analysis)，这篇博客里的内容写得挺不错的，很多内容都一针见血，准备按照博客中的思路来解读源码，不打算一步一步来了，太无聊。

### 类型判断

jQuery 里面有一个判断类型的函数，就是 `$.type`，它最主要的好处就是一个函数可以对所以的类型进行判断，然后返回类型名。`_` 中的判断略坑，函数很多，而且都是以 `is` 开头，什么 `isArray`，`isFunction`等等。

```javascript
var toString = Object.prototype.toString,
  nativeIsArray = Array.isArray;

_.isArray = nativeIsArray || function(obj) {
  return toString.call(obj) === '[object Array]';
};
```

可以看得出来，设计者的心思还是挺仔细的，当然，还有：

```javascript
_.isObject = function(obj) {
  var type = typeof obj;
  return type === 'function' || type === 'object' && !!obj;
};
_.isBoolean = function(obj) {
  return obj === true || obj === false || toString.call(obj) === '[object Boolean]';
};
```

`isObject` 的流程看起来有点和 array、boolean 不一样，但是也是情理之中，很好理解，那么问题来了，这样会不会很麻烦，光构造这些函数就要花很久的时间吧，答案用下面的代码来解释：

```javascript
_.each(['Arguments', 'Function', 'String', 'Number', 'Date', 'RegExp', 'Error'], function(name) {
  _['is' + name] = function(obj) {
    return toString.call(obj) === '[object ' + name + ']';
  };
});
```

对于一些不用特殊处理的函数，直接用 `each` 函数来搞定。

除此之外，还有一些有意思的 `is` 函数：

```javascript
// 只能用来判断 NaN 类型，因为只有 NaN !== NaN 成立，其他 Number 均不成立
_.isNaN = function(obj) {
  return _.isNumber(obj) && obj !== +obj;
};

// null 严格等于哪些类型？
_.isNull = function(obj) {
  return obj === null;
};

// 又是一个严格判断 ===
// 貌似 _.isUndefined() == true 空参数的情况也是成立的
_.isUndefined = function(obj) {
  return obj === void 0;
};
```

不过对于 `isNaN` 函数，还是有 bug 的，比如：

```javascript
_.isNaN(new Number(1)); // true
// new Number(1) 和 Number(1) 是有区别的
```

这边 github issue 上已经有人提出了这个问题，[_.isNaN](https://github.com/jashkenas/underscore/issues/2257)，也合并到分支了 [Fixes _.isNaN for wrapped numbers](https://github.com/jashkenas/underscore/pull/2259)，但是不知道为什么我这个 1.8.3 版本还是老样子，难度我下载了一个假的 underscore？issue 中提供了解决办法：

```javascript
_.isNaN = function(obj) {
  // 将 !== 换成 !=
  return _.isNumber(obj) && obj != +obj;
};
```

我跑去最新发布的 underscore 下面看了下，最近更新 `4 month ago`，搜索了一下 `_.isNaN`：

```javascript
_.isNaN = function(obj) {
  // 真的很机智，NaN 是 Number 且 isNaN(NaN) == true
  // new Number(1) 这次返回的是 false 了
  return _.isNumber(obj) && isNaN(obj);
};
```

来看一眼 jQuery 里面的类型判断：

```javascript
// v3.1.1
var class2type = {
    "[object Boolean]": "boolean",
    "[object Number]": "number",
    "[object String]": "string",
    "[object Function]": "function",
    "[object Array]": "array",
    "[object Date]": "date",
    "[object RegExp]": "regexp",
    "[object Object]": "object",
    "[object Error]": "error",
    "[object Symbol]": "symbol"
}
var toString = Object.prototype.toString;

jQuery.type = function (obj) {
    if (obj == null) {
        return obj + "";
    }
    return 
      typeof obj === "object" || typeof obj === "function" ? 
        class2type[toString.call(obj)] || "object" : 
        typeof obj;
}
```

比较了一下，发现 jQuery 相比于 underscore，少了 `Arguments` 的判断，多了 ES6 的 `Symbol` 的判断（PS：underscore 好久没人维护了？）。所以 jQuery 对 Arguments 的判断只能返回 `object`，`_` 中是没有 `_.isSymbol` 函数的。以前一直看 jQuery 的类型判断，竟然不知道 Arguments 也可以单独分为一类 `arguments`。还有就是，如果让我来选择在项目中使用哪个，我肯定选择 jQuery 的这种方式，尽管 underscore 更详细，但是函数拆分太多了。

### 其他有意思的 is 函数

前面说了，underscore 给人一种很啰嗦的感觉，`is` 函数太多，话虽如此，总有几个非常有意思的函数：

```javascript
_.isEmpty = function(obj) {
  if (obj == null) return true;
  if (isArrayLike(obj) && (_.isArray(obj) || _.isString(obj) || _.isArguments(obj))) return obj.length === 0;
  return _.keys(obj).length === 0;
};
```

`isEmpty` 用来判断是否为空，我刚开始看到这个函数的时候，有点懵，说到底还是对 `Empty` 这个词理解的不够深刻。到底什么是 **空** 呢，看源码，我觉得这是最好的答案，毕竟汇集了那么多优秀多 JS 开发者。

1. 所有与 `null` 相等的元素，都为空，没问题；
2. 数组、字符串、Arguments， 它们也可以为空，比如 length 属性为 0 的时候；
3. 最后，用自带的 _.keys 判断 obj key 集合的长度是否为 0。

有时候觉得看代码，真的是一种升华。

还有一个 `isElement`，很简单，只是不明白为什么用了两次非来判断：

```javascript
_.isElement = function(obj) {
  // !! 
  return !!(obj && obj.nodeType === 1);
};
```

重点来说下 `isEqual` 函数：

```javascript
_.isEqual = function(a, b) {
  return eq(a, b);
};

var eq = function(a, b, aStack, bStack) {
  // 解决 0 和 -0 不应该相等的问题？
  // See the [Harmony `egal` proposal](http://wiki.ecmascript.org/doku.php?id=harmony:egal).
  if (a === b) return a !== 0 || 1 / a === 1 / b;
  // 有一个为空，直接返回，但要注意 undefined !== null
  if (a == null || b == null) return a === b;
  // 如果 a、b 是 _ 对象，返回它们的 warpped
  if (a instanceof _) a = a._wrapped;
  if (b instanceof _) b = b._wrapped;
  var className = toString.call(a);
  // 类型不同，直接返回 false
  if (className !== toString.call(b)) return false;
  switch (className) {
    // Strings, numbers, regular expressions, dates, and booleans are compared by value.
    case '[object RegExp]':
    // RegExps are coerced to strings for comparison (Note: '' + /a/i === '/a/i')
    case '[object String]':
      // 通过 '' + a 构造字符串
      return '' + a === '' + b;
    case '[object Number]':
      // +a 可以将类型为 new Number 的 a 转变为数字
      // +a !== +a，只能说明 a 为 NaN，判断 b 是否也为 NaN
      if (+a !== +a) return +b !== +b;
      return +a === 0 ? 1 / +a === 1 / b : +a === +b;
    case '[object Date]':
    case '[object Boolean]':
      // +true === 1
      // +false === 0
      return +a === +b;
  }

  // 如果以上都不能满足，可能判断的类型为数组或对象，=== 是无法解决的
  var areArrays = className === '[object Array]';
  // 非数组的情况，看一下它们是否同祖先，不同祖先，failed
  if (!areArrays) {
    // 奇怪的数据
    if (typeof a != 'object' || typeof b != 'object') return false;

    // Objects with different constructors are not equivalent, but `Object`s or `Array`s
    // from different frames are.
    var aCtor = a.constructor, bCtor = b.constructor;
    if (aCtor !== bCtor && !(_.isFunction(aCtor) && aCtor instanceof aCtor &&
                             _.isFunction(bCtor) && bCtor instanceof bCtor)
                        && ('constructor' in a && 'constructor' in b)) {
      return false;
    }
  }
  // Assume equality for cyclic structures. The algorithm for detecting cyclic
  // structures is adapted from ES 5.1 section 15.12.3, abstract operation `JO`.

  // Initializing stack of traversed objects.
  // It's done here since we only need them for objects and arrays comparison.
  aStack = aStack || [];
  bStack = bStack || [];
  var length = aStack.length;
  while (length--) {
    // 这个应该是为了防止嵌套循环
    if (aStack[length] === a) return bStack[length] === b;
  }

  // Add the first object to the stack of traversed objects.
  aStack.push(a);
  bStack.push(b);

  // Recursively compare objects and arrays.
  if (areArrays) {
    length = a.length;
    // 数组长度都不等，肯定不一样
    if (length !== b.length) return false;
    // 递归比较，如果有一个不同，返回 false
    while (length--) {
      if (!eq(a[length], b[length], aStack, bStack)) return false;
    }
  } else {
    // 非数组的情况
    var keys = _.keys(a), key;
    length = keys.length;
    // 两个对象的长度不等
    if (_.keys(b).length !== length) return false;
    while (length--) {
      key = keys[length];
      if (!(_.has(b, key) && eq(a[key], b[key], aStack, bStack))) return false;
    }
  }
  // 清空数组
  aStack.pop();
  bStack.pop();
  return true; // 一路到底，没有失败，则返回成功
};
```

总结一下，就是 `_.isEqual` 内部虽然用的是 `===` 这种判断，但是对于 `===` 判断失败的情况，`isEqual` 会尝试将比较的元素拆分比较，比如，如果是两个不同引用地址数组，它们元素都是一样的，则返回 `true`：

```javascript
[22, 33] === [22, 33]; // false
_.isEqual([22, 33], [22, 33]); // true

{id: 3} === {id: 3}; // false
_.isEqual({id: 3}, {id: 3}); // true

NaN === NaN; // false
_.isEqual(NaN, NaN); // true

/a/ === new RegExp('a'); // false
_.isEqual(/a/, new RegExp('a')); // true
```

可以看得出来，`isEqual` 是一个非常有心机的函数。

### 数组去重

关于数组去重，从面试笔试的程度来说，是家常便饭的题目，实际中也会经常用到，前段时间看到一篇去重的博客，感觉含金量很高，地址在这：[也谈JavaScript数组去重](https://www.toobug.net/article/array_unique_in_javascript.html)，年代在久一点，就是玉伯大虾的[从 JavaScript 数组去重谈性能优化](https://github.com/lifesinger/blog/issues/113)。

`_` 中也有去重函数 uniq 或者 unique：

```javascript
_.uniq = _.unique = function(array, isSorted, iteratee, context) {
  // 和 jQuery 一样，平移参数
  if (!_.isBoolean(isSorted)) {
    context = iteratee;
    iteratee = isSorted;
    isSorted = false;
  }
  // 又是回调 cb，三个参数
  if (iteratee != null) iteratee = cb(iteratee, context);
  var result = [];
  var seen = [];
  for (var i = 0, length = getLength(array); i < length; i++) {
    var value = array[i],
        computed = iteratee ? iteratee(value, i, array) : value;
    // 如果已经排列好，就直接和前一个进行比较
    if (isSorted) {
      if (!i || seen !== computed) result.push(value);
      seen = computed;
    } else if (iteratee) {
      // seen 此时化身为一个去重数组，前提是有 iteratee 函数
      if (!_.contains(seen, computed)) {
        seen.push(computed);
        result.push(value);
      }
    } else if (!_.contains(result, value)) {
      result.push(value);
    }
  }
  return result;
};
```

还是要从 `unique` 的几个参数说起，第一个参数是数组，第二个表示是否已经排好序，第三个参数是一个函数，表示对数组的元素进行怎样的处理，第四个参数是第三个参数的上下文。返回值是一个新数组，思路也很清楚，对于已经排好序的数组，用后一个和前一个相比，不一样就 push 到 result 中，对于没有排好序的数组，要用到 `_.contains` 函数对 result 是否包含元素进行判断。

去重的话，如果数组是排好序的，效率会很高，时间复杂度为 n，只要遍历一次循环即刻，对于未排好序的数组，要频繁的使用 `contains` 函数，复杂度很高，平均为 n 的平方。去重所用到为相等为严格等于 `===`，使用的时候要小心。

`_.contains` 函数如下所示：

```javascript
_.contains = _.includes = _.include = function(obj, item, fromIndex, guard) {
  if (!isArrayLike(obj)) obj = _.values(obj);
  if (typeof fromIndex != 'number' || guard) fromIndex = 0;
  return _.indexOf(obj, item, fromIndex) >= 0;
};

_.indexOf = createIndexFinder(1, _.findIndex, _.sortedIndex);
_.lastIndexOf = createIndexFinder(-1, _.findLastIndex);

function createIndexFinder(dir, predicateFind, sortedIndex) {
  return function(array, item, idx) {
    var i = 0, length = getLength(array);
    if (typeof idx == 'number') {
      if (dir > 0) {
          i = idx >= 0 ? idx : Math.max(idx + length, i);
      } else {
          length = idx >= 0 ? Math.min(idx + 1, length) : idx + length + 1;
      }
    } else if (sortedIndex && idx && length) {
      idx = sortedIndex(array, item);
      return array[idx] === item ? idx : -1;
    }
    // 自己都不等于自己，让我想到了 NaN
    if (item !== item) {
      idx = predicateFind(slice.call(array, i, length), _.isNaN);
      return idx >= 0 ? idx + i : -1;
    }
    for (idx = dir > 0 ? i : length - 1; idx >= 0 && idx < length; idx += dir) {
      // 这里使用的是严格等于
      if (array[idx] === item) return idx; // 找到，返回索引
    }
    return -1; // 没找到，返回 -1
  };
}
```

## 总结

感觉 `Underscore` 的源码看起来还是很简单的，Underscore 里面有一些过时的函数，这些都可以拿过来学习，逻辑比较清晰，并不像 jQuery 那样，一个函数里面好多内部函数，看着看着就晕了。

## 参考

>[Underscore.js (1.8.3) 中文文档](http://www.css88.com/doc/underscore/)

>[常用类型判断以及一些有用的工具方法](https://github.com/hanzichi/underscore-analysis/issues/2)

>[也谈JavaScript数组去重](https://www.toobug.net/article/array_unique_in_javascript.html)

>[JavaScript 数组去重](https://github.com/hanzichi/underscore-analysis/issues/9)