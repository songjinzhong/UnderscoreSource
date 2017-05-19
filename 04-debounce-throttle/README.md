来聊聊 `_` 中的去抖动函数和节流函数，它们经常出现在高频函数中，在不影响体验到情况下，可以提升浏览器的性能。

## 我们为什么需要防抖动和节流

防抖动函数和节流函数是两个不同的函数，但是也有一定的联系，选择的时候有一定的针对性。所谓的防抖动，是指一些高频函数执行的次数太多了，通过添加一个延迟，假如是 100ms，表示**高频函数触发后并不立即执行，100ms 后执行，如果高频函数在此期间再次触发，则重新计时。**所谓的节流，就是采用防抖动的策略，但是**设定一个时间，在这个时间内高频函数必须要执行一次。**节流和去抖都采用延迟时间去执行函数，而节流又设定了一个频率时间，在这个频率时间内，函数必须执行一次（也可以理解为函数执行的频率，这和节流的本意很相似。）

那么，什么时候用节流，什么时候用去抖呢？

先来看下应用场景，在浏览器中，有如下的场景，window 的 `resize` 和 `scroll`事件，鼠标的 `mousemove` 事件，触摸屏的 `touchmove` 事件，这些事件都可以称为高频函数。比如 `scroll` 事件：

```javascript
// 要考虑浏览器兼容性问题，在 chrome 下测试
window.onscroll = function(){
  console.log('scroll');
}
```

轻轻滚动浏览器，就可以计算出 onscroll 事件执行的次数。

如果对于高频函数，我们只是想延迟它的执行，在保证一定会执行一次的情况下，从高频变成低频，就可以使用去抖函数。比如 window 的 `resize` 事件，我们不关心 resize 的过程，只关心 resize 执行后，根据当前的 window 尺寸进行相应的修改。如果想要将高频函数变成低频函数，在固定的频率内执行，使用节流函数。比如鼠标的拖拽事件，使用去抖会导致当拖拽的频率很快，函数会“假死”，这时候应该用节流，将函数的执行频率从原本每秒 100 次降低为每秒 20次（当然，这只是假设）。

又如 `scroll` 事件，如下图所示，是我博客中的一个文章目录，当我滚动的时候，浏览器能够识别当前进入到屏幕中的标题是哪个，并能够设置高亮提醒。

![](http://omuedc5n9.bkt.clouddn.com/test.gif)

较早的时候，我用的是去抖实现来优化 scroll 事件，后来发现当鼠标滚动非常快的时候，是看不到这个渐变的过程，而是一个突变的过程。后来改成节流之后，效果明显变好了。

## 去抖函数的实现

去抖函数的应用场景如下：

1. 用于 resize、scroll 函数，只计算最后一次的触发，比如当窗口大小变化时，重新计算布局；
2. 输入框发送 ajax 请求或者两个 input 框之间的映射；

去抖函数的实现可以参考如下的代码：

```javascript
function debounce(fn, delay){
  var timer = null,
    self = this;
  return function(){
    var args = arguments;
    clearTimeout(timer);
    timer = setTimeout(function(){
      fn.apply(self, args);
    }, delay)
  }
}
```

这是一个非常机智的做法，此方法在其他地方也很常见。用闭包保留当前的上下文，定义一个全局的 timer，每一次函数执行的时候，并不是直接执行，先清除上一次 Timeout，然后延迟 delay 执行。这样做的好处，很多高频函数就可以减少触发。

保留了上下文，这样就可以指定 fn 的上下文：

```javascript
var obj = { id: 0 };
var print = function(){
  console.log(this.id ++);
}
// 绑定上下文，延迟 200ms 执行
window.onresize = debounce.call(obj, print, 200);
```

如果 `onresize` 的速度很快，只有在最后一次，打印一次 id，这就是去抖。

## 节流函数的实现

节流函数的应用场景要更多一些：

1. canvas 的 touchmove
2. 鼠标的 mousemove
3. 连续的 keyup、keydown、mouseup、mousedown 事件
4. scroll 事件，需要经常处理，比如图片的滚动加载

首先，节流是一个去抖函数，只不过加入了一个频率时间：

```javascript
function throttle(fn, delay, mustRun){
  var timer, startTime = new Date(), self = this;
  return function(){
    var curTime = new Date(), args = arguments;
    clearTimeout(timer);
    if(curTime - startTime >= mustRun){
      startTime = curTime;
      fn.apply(self, args);
    }else{
      timer = setTimeout(function(){
        fn.apply(self, args);
      }, delay)
    }
  }
}
```

`throttle` 有三个参数，第一个参数和第二个参数分别是回调函数和延迟时间，第三个参数代表如果函数在 mustRun 之后还没有执行，则必须执行一次。通过逻辑可以发现，如果将 mustRun 设置成非常大的值，比如十分钟，`throttle` 和 `debounce` 效果一样。

```javascript
var obj = { id: 0 };
var print = function(){
  console.log(this.id ++);
}
window.onscroll = throttle.call(obj, print, 200, 50);
```

函数执行的频率被限制了，函数执行的最短时间为 50ms（理想情况下），频率的最大值为每秒 20次，这就是节流函数。

## `_` 中的节流和去抖

在 `_` 中，有专门针对于节流和去抖的函数，分别是 `_.debounce` 和 `_.throttle`，不介绍了，直接进入源码吧。

### 去抖函数源码

去抖的实现，采用的方式和我之前介绍的完全不一样，但思路都是延迟和闭包，去抖函数还有一个隐藏属性，如果开启第三个参数，则会使得去抖函数变成立即执行一次函数，函数触发后会立即执行一次，且只会执行一次，除非本轮循环结束。立即执行函数也非常有用，比如鼠标 click 事件，需要立即执行，且要防止用户短时间内点击两次。

```javascript
_.debounce = function(func, wait, immediate) {
  var timeout, args, context, timestamp, result;

  // later 函数是延迟后执行的函数
  var later = function() {
    var last = _.now() - timestamp; // 获取延迟时间

    if (last < wait && last >= 0) {
      timeout = setTimeout(later, wait - last); // 继续延迟
    } else {
      timeout = null; //执行，设置 null 开启下次循环
      if (!immediate) {
        result = func.apply(context, args);
        if (!timeout) context = args = null;
      }
    }
  };

  return function() {
    context = this; // 上下文和参数
    args = arguments;
    timestamp = _.now(); // 记录当前的时间点
    var callNow = immediate && !timeout; // 判断是否开启立即函数
    if (!timeout) timeout = setTimeout(later, wait); // timeout 开始计时
    // 立即执行
    if (callNow) {
      result = func.apply(context, args);
      context = args = null;
    }

    return result;
  };
};
```

## 参考

>[关于js函数节流和去抖动](http://www.jianshu.com/p/4f3e2c8f5e95)

>[[JavaScript] 函数节流与去抖](https://github.com/hahnzhu/read-code-per-day/issues/5)

>[JavaScript 函数节流和函数去抖应用场景辨析](https://github.com/hanzichi/underscore-analysis/issues/20)