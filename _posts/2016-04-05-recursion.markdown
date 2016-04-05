---
layout:   post
title:    "谈谈递归"
author:   WUKL
comments: true
---

### 什么是递归

小时候，我们都听过下面这则故事：

> 从前有座山，山里有个庙，庙里有个老和尚在给小和尚讲故事，讲的什么呢？「从前有座山，山里有个庙，庙里有个老和尚在给小和尚讲故事，讲的什么呢？『从前有座山，山里有个庙，庙里有个老和尚在给小和尚讲故事，讲的什么呢？……』」

这个故事自己套着自己，没完没了，像是某种特征在不断地重复，这就是递归了。

留心观察一下我们身边的世界，其实递归无处不在，[GNU](https://www.gnu.org/pronunciation/pronunciation.html)、[PHP](http://php.net/manual/en/intro-whatis.php)，[多肉植物](http://huaban.com/pins/670977064/zoom/)、[松塔的螺线](http://huaban.com/pins/632274394/zoom/)、[向日葵的花盘](http://huaban.com/pins/670977260/zoom/)、[蕨类植物的叶子](http://huaban.com/pins/632273978/zoom/) ，[平行镜中的像](http://huaban.com/pins/670977618/zoom/)、[拍自己的摄像机](https://youtu.be/OWnC9tSA3iA)等等，都有某种自相似的特征。

在计算机科学里，[递归](https://zh.wikipedia.org/wiki/%E9%80%92%E5%BD%92_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6))指的是一种通过重复将问题分解为同类的子问题而解决问题的方法。套用程序的概念，就是函数执行过程中直接或间接地调用自身。

### 用递归来解决问题

由递归的概念可知，如果某个问题有一些自相似的特征，可以分解为子问题，那么它就可以被递归定义。如果问题被递归定义了，也差不多解决了。

我们来看一个具体例子：


假设现在有 5 个人、5 个座位，那这几个人的位置分布情况有几种。我们先来列举一下：

```
第 1 个人： 有 5 种选择
第 2 个人： 有 (5 - 1) 种选择
第 3 个人： 有 (5 - 2) 种选择
第 4 个人： 有 (5 - 3) 种选择
第 5 个人： 有 (5 - 4) 种选择
```

那么 5 个人总共的选择数是：

```
5 * (5 - 1) * (5 - 2) * (5 - 3) * (5 - 4) 
```

同理，如果总共是 n 个人、n 个座位，那么所有可能的情况数 f(n) 可以表示为：

```
f(n) = n * (n - 1) * … * 2 * 1
```

这里的未知数 n 有无数种可能，我们不可能把所有的情况都列举出来，然后得到结果。要解决这样的问题，就要运用递归的思想，找到各个情况的共同模式，切分成小的有限的问题，然后尝试着把它描述出来。

我们先把 n <= 5 的情况列举出来：

```
f(1) =                 1
f(2) =             2 * 1
f(3) =         3 * 2 * 1
f(4) =     4 * 3 * 2 * 1
f(5) = 5 * 4 * 3 * 2 * 1
```

通过对比可以观察到一个现象：

```
f(1) = 1
f(2) = 2 * f(1)
f(3) = 3 * f(2)
f(4) = 4 * f(3)
f(5) = 5 * f(4)
```

我们找到了一个共同模式，描述出来就是，对于 f(n) 来说，它的值是参数 n 和前一步的值的乘积，前一步和后一步的参数之间相差 1，所以我们很自然就想到用这个 1 做切分，即：

```
f(n) = n * f(n - 1)
```

这样我们就定义了这个函数 f，它会在执行的过程中调用自己，又称为递归函数，用 JavaScript 实现就是这样：

```javascript
var f = function(n) {
  return n * f(n - 1);
}
```

简单分析一下，这个函数每执行一次，参数 n 就少 1，最终它会到达 0，之后还会进一步到负数，看起来好像永远都得不到结果。

我们回到命题，当 n 到达 0 时，`f(0)` 指的就是「 在 0 个人、0 个座位时的位置分布情况」，没有意义，是 [Empty product](https://en.wikipedia.org/wiki/Empty_product)，值为 1。而 n 是负数时就更没有意义了，所以 n 应该是一个自然数：

```
f(0) = 1
f(n) = n * f(n - 1)
```

这里的 n = 0 就是递归函数 f 的一个临界条件（edge case），当达到临界条件时，递归函数继续执行没有意义，此时递归结束。在定义递归函数的时候，设定合适的临界条件很重要，否则容易陷入死循环。

相应的，我们的代码也要改变：

```javascript
// 参数 n 是自然数，后文不再说明
var f = function(n) {
  if (n === 0) return 1;
  return n * f(n - 1);
}
```

如果不用 `if-else` `switch` `? :` 之类的条件判断语句，能不能实现类似的效果呢？

```javascript
var f = function(n) {
  return [1][n] || n * f(n - 1);
}
```

由此，我们就通过定义递归函数 f 描述了这个问题，对于任意的自然数 n，我们只需执行 `f(n)` 就可以得到答案。

用递归的思路来解决问题，并不是写出具体的求解步骤，而是试着把问题描述出来，考虑有哪些共同模式，怎么切分，何处结束，以及何处执行递归。  



### 递归函数的优化

当 n = 5 时，我们把上面这个函数完整的执行过程模拟出来：

```
f(5)
5 * f(4)
5 * (4 * f(3))
5 * (4 * (3 * f(2)))
5 * (4 * (3 * (2 * f(1))))
5 * (4 * (3 * (2 * (1 * f(0)))))
5 * (4 * (3 * (2 * (1 * 1))))
5 * (4 * (3 * (2 * 1)))
5 * (4 * (3 * 2))
5 * (4 * 6)
5 * 24
120
```

我们可以看到，随着 f(5) 这个函数的执行，需要记录的中间状态的数目一直在变，先增后减。在空间消耗上，表现就是栈先累积后收缩，整个计算过程空间复杂度是 O(n)，当 n 很大的时候会栈溢出（[Stack overflow](https://en.wikipedia.org/wiki/Stack_overflow)）。


我们思考一下另一种方案，刚才的执行是从 n 开始，一步步计算，直到临界条件 0 为止。现在试着倒过来，由 1 开始，一步步计算，直到 n 为止，当然，此时的临界条件也相应的变为「超过 n」。


实现起来应该就是这样：

```javascript
var f = function(n) {
  return f2(n, 1, 1);
};

var f2 = function(n, i, result) {
  if (i > n) {
    return result;
  } else {
    return f2(n, i + 1, result * i);
  }
};
```

我们引入了另一个函数 f2 来完成从 1 到 n 的递归。函数 f2 在尾位置（也就是函数执行的最后）调用自身，这样的递归称为尾递归（Tail recursion）。

不少编程语言（包括 ES6）都支持对这样的函数进行空间优化，也叫尾递归优化（[Tail-call optimization](https://en.wikipedia.org/wiki/Tail-call_optimization)）。

具体是怎么进行尾递归优化的呢？

我们回顾一下函数 f2 的定义，因为函数 f2 每次执行的最后是函数调用，而下一步执行所需要的状态都是通过参数传递的，那么当前栈就可以清空被重用。比如，在计算第 3 步的时候，之前的第 1 步、第 2 步的中间状态都不需要了，只保留第 3 步执行需要的参数就行。

我们把优化后的步骤列出来：

```
f(5)
f2(5, 1, 1)
f2(5, 2, 1)
f2(5, 3, 2)
f2(5, 4, 6)
f2(5, 5, 24)
f2(5, 6, 120)
120
```

可以看出，这种方案随着计算的增加，消耗的空间一直不变，占用恒量的内存，和迭代程序一样，它的空间复杂度是 O(1)。

所以经过尾递归优化后，使得递归计算可以跟 `while`、`for-loop` 等迭代式计算的效率基本相当。

### 递归和不动点组合子

我们再来看一下目前的递归函数：

```javascript
var f = function(n) {
  return [1][n] || n * f(n - 1);
}
```

它是通过在具名函数 f 体内，引用函数定义的变量名 f 来实现自我调用，我们只有在运行时才知道 f 是什么，这就带来了两个问题：

1. 函数定义的变量名，是一个自由变量，很容易被改变，当然这个可以通过闭包的方式解决。
2. 不适用于匿名函数。

我们需要另一种在函数体内完成自我调用的方法，即找到下面这个函数里的 self（注：`arguments.callee` [在 ES5 严格模式下不可用](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/arguments/callee) ）。

```javascript
function(n) {
  return [1][n] || n * self(n - 1);
}

// self = ?
```

既然不能使用自由变量的方式调用自身，我们可以选择约束变量，即把自身 self 通过参数的方式传进去。那么就会变成这样：

```javascript
// 先声明我们需要的目标函数 fact
// fact(5) => 120
var fact;

// 这里引入了一个高阶函数 f，接收函数 self 作为参数
// self 是指向目标函数自身的函数，也就是 fact
// self = fact
var f = function(self) {
  return function(n) {
    return [1][n] || n * self(n - 1);
  };
};

fact = f(self);
```

因为 self = fact、fact = f(self)，所以：

```
self = f(self)
```

我们先暂停一下，打开计算器，随便输入一个数字，一直按 cos 键，经过 `cos(cos(...cos(x)))` 之后，输出结果会稳定于值 v ，也就是说：

```
v = cos(v)
```

此时，函数的参数和返回值相同，我们称这个值是这个函数的不动点（[Fix-point](https://en.wikipedia.org/wiki/Fixed_point_(mathematics))）。

回到这个例子，我们现在知道 self 就是高阶函数 f 的不动点。

于是我们的问题就变成了找出函数 f 的不动点。

假设存在一个函数 Y，我们把函数 f 作为参数传给它后，它可以返回 f 的不动点 self，即：

```
self = Y(f)
```

前面我们已经知道，self 指向的就是 fact，那我们的目标函数 fact 也就可以通过 Y 得到：

```
fact = Y(f)
```

现在我们的问题变成了，找到这个 Y。

好，下面我们就从最初的函数起，一步一步推导出这个 Y。先来回顾一下最初的函数：

```javascript
function(n) {
  return [1][n] || n * self(n - 1);
}
```

这里的 self 指向其所在函数自身，因为是匿名函数，没法直接引用定义，需要通过参数的方式传进去：

```javascript
function(n, self) {
  return [1][n] || n * self(n - 1, self);
}
```

现在我们给这个函数绑定一个名字 f，这样就变成了：

```javascript
var f = function(n, self) {
  return [1][n] || n * self(n - 1, self);
};

f(5, self)
```

刚才我们已经知道 self 指向的是自身，也就是现在的绑定名 f，即 self = f，因此我们在 f 调用的地方可以直接用 f 代替参数 self 传给 `f(5, self)` ，即：

```javascript
var f = function(n, self) {
  return [1][n] || n * self(n - 1, self);
};

f(5, f) // => 120
```

现在这个函数虽然可用，但变成了一个二元函数，我们要的其实是一元函数，所以要做柯里化（[Currying](https://en.wikipedia.org/wiki/Currying)）。

先来看一下下面这个小例子，简单了解一下什么是柯里化：

```javascript
var foo = function(x, y) {
  return x + y;
};
foo(1, 2) // => 3

foo = function(y) {
  return function (x) {
    return x + y;
  };
};
foo(2)(1) // => 3

// 像上面这样从 foo(x, y) 到 foo(y)(x) 的变化就是柯里化
```

现在对函数 f 做柯里化：

```javascript
var f = function(self) {
  return function(n) {
    // 此时 self 是指向函数 f
    // 原来的 f 柯里化后调用方式变成了 f(f)(n)
    // 所以 self 也要相应的变为 self(self)(n - 1)
    return [1][n] || n * self(self)(n - 1);
  };
};

f(f)(5) // => 120
```

现在的问题是我们的函数里混杂了 `self(self)` ，应该将它拿到函数外部，通过参数的方式传给 f：

```javascript
var g = function(h) {
  // 把原来 f 内部的 self(self) 拿出来，放到新声明的函数 x 里
  // 此时 x(n) 应该等于原来的 self(self)(n)
  // 再把 x 作为参数传回给 f
  var x = function(n) {
    return h(h)(n);
  };

  // 此时，函数 f 变回了最初的形式
  // f 的参数 self 也相应的变回了指向函数 f 的不动点
  var f = function(self) {
    return function(n) {
      return [1][n] || n * self(n - 1);
    };
  };

  return f(x);
};

g(g)(5) // => 120
```

现在我们可以把 f 拿出来放到函数 g 的外面：

```javascript
var f = function(self) {
  return function(n) {
    return [1][n] || n * self(n - 1);
  };
};

var g = function(h) {
  var x = function(n) {
    return h(h)(n);
  };
  return f(x);
};

var fact = g(g);

fact(5) // => 120
```

函数 g 里面的 f 是一个自由变量，我们再做转化，让 f 也成为一个约束变量，即通过参数的方式传递。这就需要把函数 g 和 fact 定义的部分（即 `g(g)`）一起放入闭包 wrap 里：

```javascript
var f = function(self) {
  return function(n) {
    return [1][n] || n * self(n - 1);
  };
};

var wrap = function(f) {
  var g = function(h) {
    var x = function(n) {
      return h(h)(n);
    };
    return f(x);
  };
  return g(g);
};

var fact = wrap(f);

fact(5) // => 120
```

这样转化后得到的函数 wrap，其实就是我们要找的 Y，把它单独拿出来：

```javascript
var Y = function(f) {
  var g = function(h) {
    var x = function(n) {
      return h(h)(n);
    };
    return f(x);
  };
  return g(g);
};
```

精简一下，去掉所有的中间变量：

```javascript
var Y = function(f) {
  return (function(g) {
    return g(g);
  })(function(h) {
    return f(function(n) {
      return h(h)(n);
    });
  });
};
```

用函数 f 测试一下吧：

```javascript
var f = Y(function(self) {
  return function(n) {
    return [1][n] || n * self(n - 1);
  };
});

f(5) // => 120
```

但现在的它还有一个小缺点，就是只支持一个参数，我们再稍微改一下，让它支持任意参数：

```javascript
var Y = function(f) {
  return (function(g) {
    return g(g);
  })(function(h) {
    return f(function() {
      return h(h).apply(null, arguments);
    });
  });
};
```

好了，这就是最终版 Y，我们拿之前实现的尾递归函数测试一下：

```javascript
var f = Y(function(self) {
  return function(n, i, result) {
    if (i > n) {
      return result;
    } else {
      return self(n, i + 1, result * i);
    }
  };
});

var fact = function(n) {
  return f(n, 1, 1);
};

fact(5) // => 120
```

那现在我们就找到了这个 Y，它的正式名字叫 Y 组合子（Y combinator），最初被 [Haskell Curry](https://zh.wikipedia.org/wiki/Haskell_Curry) 发现，用于 [λ 演算](http://en.wikipedia.org/wiki/Lambda_calculus) 。不过我们找到的这个 Y 和 Haskell Curry 最初发现的有点不一样，我们实现的是适用于 JavaScript 这种传值调用的语言，也叫做 Z 组合子，它还有一个更接近最初版的变体是这个样子的：

```javascript
var Z = function(f) {
  return (function(g) {
    return function(x) {
      return f(g(g))(x);
    };
  })(function(g) {
    return function(x) {
      return f(g(g))(x);
    };
  });
}
```

Y 组合子是不动点组合子（[Fix-point combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator)）的一个实现，用于计算高阶函数的不动点。不动点组合子避免了把函数定义混淆在函数自身的命名空间内，使得程序执行过程更加流式（streamable）。它适用于任何支持匿名函数的语言，在匿名函数体内，因为没法引用到函数定义自身，没法递归，但有了不动点组合子后，我们只需把这个匿名函数稍加变化，作为参数传给它，就可以定义递归函数了。

### 递归和自产生程序

艾舍尔（M. C. Escher）著名的作品[《画手》](http://www.wikiart.org/en/m-c-escher/drawing-hands) ，看起来就像是自己在画自己。那是否也有这样一个程序，自己打印出自己呢？

我们还是以 JavaScript 作为演示语言，先来看下打印的方法：

```javascript
console.log('hello')
// => hello
```

接着试着打印出上面这条语句：

```javascript
console.log("console.log('hello')")
// => console.log('hello')
```

由此可以看到，虽然已经可以打印出程序，但打印出来的只是部分程序（只有参数部分），我们的目标是打印出整个程序。

仔细观察一下，这里存在着某种自相似的特征，即 `打印"打印'打印'"` 这样的形式，也就是说，可以用递归的思路来解决。但这里并不像上面这个例子这么直观地得到递归定义，我们先尝试模拟一下这种自相似结构，构造一个打印函数 print，在调用的时候把函数 print 作为参数传回给自身：

```javascript
// 参数 s 是一个函数，打印前先转化成 String
function print(s) { console.log(s.toString()) }
print(print)
// => function print(s) { console.log(s.toString()) }
```

但现在有两条语句，我们用函数体代替函数名作为参数，从而合并为一条：

```javascript
(function (s) { console.log(s.toString()) })(function (s) { console.log(s.toString()) })
// => function (s) { console.log(s.toString()) }
```

看来这样只打印出一半，我们稍作变化，让它打印出两半：

```javascript
(function (s) { console.log(s.toString() + s.toString()) })(function (s) { console.log(s.toString() + s.toString()) })
// => function (s) { console.log(s.toString() + s.toString()) }function (s) { console.log(s.toString() + s.toString()) }
```

在 Javascript 里，操作符 `+` 只接受数值类型（Number）或字符串类型（String）的操作数，当作用于其他类型时，会先调用 `valueOf()` 或 `toString()` 进行转化。在这里，参数 `s` 是函数，`s + s` 会默认调用 `s.toString()` ，所以上面的语句可以简化为：

```javascript
(function (s) { console.log(s + s) })(function (s) { console.log(s + s) })
// => function (s) { console.log(s + s) }function (s) { console.log(s + s) }
```

看起来已经很接近了，现在只需要把前后缺少的小括号加上去：

```javascript
(function (s) { console.log( '(' + s + ')(' + s + ')' )})(function (s) { console.log( '(' + s + ')(' + s + ')' )})
// => (function (s) { console.log( '(' + s + ')(' + s + ')' )})(function (s) { console.log( '(' + s + ')(' + s + ')' )})
```

可以看到，我们成功地打印出了自身，而这个打印自身的程序，有个正式名字叫自产生程序（[Quine](https://en.wikipedia.org/wiki/Quine_(computing))，以逻辑学家 Willard van Orman Quine 命名），顾名思义，就是把程序自身的源码打印出来。

事实上，只要是具备打印功能的图灵完备语言都有自己的 Quine 程序，而且各种各样，几乎有无限多的版本。因为简单优雅的 Quine 程序往往需要用到一些语言技巧，所以找到这样的 Quine 也是熟悉语言的好方法。  

<br />

最后，看完文章，对递归有没有比较清晰的认识呢，有什么想法，给我们留言吧。
