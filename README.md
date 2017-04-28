# UnderscoreSource

版本 v1.8.3

如果说 jQuery 和 Underscore 有什么不同的话，除了它们的作者不同之外，jQuery 的功能主要是在于 js 和 html 交互，很多操作都是针对于 dom 来的，考虑到了浏览器兼容性，而且封装了 ajax，使得开发变得更加简洁。而 underscore 可以看成是一个 js 的函数库，其中封装了很多 js 方法，如果早几年看的话，还好，现在这些方法都被融合到 es5 或 es6 中。

那么研究 underscore 的源码还有意义吗？

一个函数库，从它的设计到实现，每一个功能函数，都集中了千百人的智慧，看一下又何妨呢？

## 目录

- Directory
  + [x] [总体架构](https://github.com/songjinzhong/UnderscoreSource/tree/master/01-main)- 只有弄懂整体架构，后面的学习才好办
  + [x] [常用思路和类型判断](https://github.com/songjinzhong/UnderscoreSource/tree/master/02-exact)- 介绍一些 Underscore 中常用的思路，还有类型判断的方法、有趣的 is 函数和数组去重。

下面的文章或系列文章，在我看源码的过程中，对我帮助很大，感谢！

>[underscorejs 官网](http://underscorejs.org/)

>[Underscore.js 源码解读 & 系列文章](https://github.com/hanzichi/underscore-analysis)

>[Underscore.js (1.8.3) 中文文档](http://www.css88.com/doc/underscore/)

>[Underscore.js 阮一峰](http://javascript.ruanyifeng.com/library/underscore.html)