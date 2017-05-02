这是第三篇关于 Underscore 的源码解读，最近一段时间学的东西很少，自己太忙了，一方面忙着找实习，晚上回去还要写毕业论文。毕业论文真的很忧伤，因为是两年半，九月份就要交一个初稿，一般都是暑假写，如果暑假出去实习，是没时间点，所以要现在写一个版本出来。

## 随机洗牌算法

说实话，以前理解数组的排序，都是将数组按照一定的逻辑（由大到小或者由小到大）排序，我自己是没有碰到过随机打乱数组排序的问题。今天看到这个问题，先是一愣，为什么要把一个数组给随机乱序，仔细想想这种应用场合还是很多的，比如随机地图、随机洗牌等等，都是随机乱序的应用。

如果这是一个面试题，**将一个数组随机乱序**，我的解决办法如下：

```javascript
function randomArr( arr ){
  var ret = [],
    rand,
    len,
    newA = arr.slice(); // 为了不破坏原数组
  while(len = newA.length){
    rand = Math.floor(Math.random() * len);
    ret.push(newA.splice(rand, 1)[0]);
  }
  return ret;
}
```

这是一个解决办法，但是却不是一个高效的解决办法，首先，空间复杂度来讲，新建了两个数组（若不考虑对原数组的改变，可以只用一个返回数组），如果能在原数组上直接操作，那真的是太好了，其次时间复杂度来讲，`splice` 函数要对数组进行改变，复杂度可以算作 n，所以总的时间复杂度应该为 `O(n^2)`。

然后 `_` 里用的是所谓的[洗牌算法](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)，很高效。洗牌算法的思路有个很大的不同，用交换数组元素的方式替换原来的 `splice`，因为 splice 太坑了，然后对上面的代码进行改进：

```javascript
function randomArr2( arr ){
  var rand, temp;
  for(var i = arr.length - 1; i >= 0; i--){
    rand = Math.floor(Math.random() * ( i + 1 ));

    // 交互 i 和 rand 位置的元素
    temp = arr[i];
    arr[i] = arr[rand];
    arr[rand] = temp;
  }
  return arr;
}
```

改进后看起来就好多了，也比较好理解，还有一个循环的方式是从 0 到 n，randmo 函数没次取值到范围为 `i~n`，方法也大体是相同的。然而在 `_` 中的 `shuffle` 方法的循环却是从左到右，看了半天才明白，代码如下：

```javascript
  // [Fisher-Yates shuffle](http://en.wikipedia.org/wiki/Fisher–Yates_shuffle).
  _.shuffle = function(obj) {
    var set = isArrayLike(obj) ? obj : _.values(obj);
    var length = set.length;
    var shuffled = Array(length); // 新建一个 kength 长度的空数组
    for (var index = 0, rand; index < length; index++) {
      rand = _.random(0, index); // 获取 0～index 之间的随机数

      // shuffled[index] 是 undefined 的，先将 index 处的值替换成 rand 处的值
      // 再将 rand 处的值替换成 set 集合处的 index
      if (rand !== index) shuffled[index] = shuffled[rand];
      shuffled[rand] = set[index];
    }
    return shuffled;
  };
```

`_.shuffle` 使用的是从左到右的思路，以至于我都无法判断出这是不是随机数组，向右前进一位，找到 `0 ~ index` 之间的一个随机数，插入新元素，如果还按照之前的解题思路，在原数组的基础上进行修改，则可以得到：

```javascript
function randomArr3(arr){
  var rand, temp;
  for(var i = 0; i < arr.length; i++){
    // 这里的 random 方法和 _.random 略有不同
    rand = Math.floor(Math.random() * ( i + 1 ));
    if(rand !== i){
      temp = arr[i];
      arr[i] = arr[rand];
      arr[rand] = temp;
    }
  }
  return arr;
}
```

`_.random` 方法是一个非常实用的方法，而我在构造 `randomArr3` 函数的时候，是想得到一个 `0 ~ i` 之间的一个随机数，来看看 `_.random` 是如何方便的实现的：

```javascript
_.random = function(min, max) {
  if (max == null) {
    max = min;
    min = 0;
  }
  // 和我最终的思路是一样的
  return min + Math.floor(Math.random() * (max - min + 1));
};
```

## group 原理

有时候需要对从服务器获得的 ajax 数据进行统计、分组，这个时候就用到了 `_` 中的 group 函数。

与 group 相关的函数有三个，它们分别是 `groupBy`，`indexBy`，`countBy`，具体使用可以参考下面的代码：

```javascript
// groupBy 用来对数据进行分组
_.groupBy([3, 4, 5, 1, 8], function(v){ return v % 2 });
// {0:[4,8],1:[3,5,1]}

// 还可以用数据的属性来区分
_.groupBy(['red', 'yellow', 'black'], 'length');
// {3:["red"],5:["black"],6:["yellow"]}

// 从某一个属性入手，相同的会替换
_.indexBy([{id: 1,name: 'first'}, {id: 2, name: 'second'}, {id: 1, name: '1st'}], 'id');
// {1:{id:1,name:"1st"},2:{id:2,name:"second"}}

// 用来统计数量
_.countBy([3, 4, 5, 1, 8], function(v){ return v % 2 == 0 ? 'even' : 'odd' });
// {odd: 3, even: 2}
```

这三个函数的实现都非常类似，在 `_` 中有进行优化，即常见的函数闭包：

```javascript
var group = function(behavior) {
  return function(obj, iteratee, context) { // 函数闭包
    var result = {}; // 要返回的对象
    iteratee = cb(iteratee, context);
    _.each(obj, function(value, index) {
      var key = iteratee(value, index, obj);
      // 针对 group、index、count，有不同的 behavior 函数
      behavior(result, value, key);
    });
    return result;
  };
};

_.groupBy = group(function(result, value, key) {
  // 每个属性都是一个数组
  if (_.has(result, key)) result[key].push(value); else result[key] = [value];
});

_.indexBy = group(function(result, value, key) {
  // 对于相同的前者被后者替换
  result[key] = value;
});

_.countBy = group(function(result, value, key) {
  // 每个属性是数量
  if (_.has(result, key)) result[key]++; else result[key] = 1;
});
```

对于 `groupBy` 和 `countBy` 处理模式几乎是一摸一样的，只是一个返回数组，一个返回统计值而已。

关于统计，这三个函数用的还真的不是很多，循环很好构造，处理函数也很好构造，如果数据是数组，直接 `forEach` 循环一遍添加一个处理函数就 ok 了，不过 `_` 最大的优点就是省事吧，这种重复利用函数、函数闭包的思路，是值的借鉴的。

## 参考

>[Fisher–Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)

>[JavaScript 数组乱序](https://github.com/hanzichi/underscore-analysis/issues/15)

>[Underscore.js (1.8.3) 中文文档](http://www.css88.com/doc/underscore/)

>[浅谈 underscore 内部方法 group 的设计原理](https://github.com/hanzichi/underscore-analysis/issues/16)