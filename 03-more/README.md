这是第三篇关于 Underscore 的源码解读，最近一段时间学的东西很少，自己太忙了，一方面忙着找实习，晚上回去还要写毕业论文。毕业论文真的很忧伤，因为是两年半，九月份就要交一个初稿，一般都是暑假写，如果暑假出去实习，是没时间点，所以要现在写一个版本出来。

## 洗牌算法

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

改进后看起来就好多了，也比较好理解，还有一个循环的方式是从 0 到 n，randmo 函数没次取值到范围为 `i~n`，方法也大体是相同的。然而在 `_` 中的 `shuffle` 方法的循环却非常奇葩，看了半天才明白，代码如下：

```javascript
  // [Fisher-Yates shuffle](http://en.wikipedia.org/wiki/Fisher–Yates_shuffle).
  _.shuffle = function(obj) {
    var set = isArrayLike(obj) ? obj : _.values(obj);
    var length = set.length;
    var shuffled = Array(length);
    for (var index = 0, rand; index < length; index++) {
      rand = _.random(0, index);
      if (rand !== index) shuffled[index] = shuffled[rand];
      shuffled[rand] = set[index];
    }
    return shuffled;
  };
```

## 参考

>[Fisher–Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle)

>[JavaScript 数组乱序](https://github.com/hanzichi/underscore-analysis/issues/15)

>[Underscore.js (1.8.3) 中文文档](http://www.css88.com/doc/underscore/)