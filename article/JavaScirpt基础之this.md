# JavaScirpt基础之this

## 写在前面
本片文章是对《你不知道的JavaScript》中this的总结笔记

在面向对象语言中this出现在calss中，在JavaScript中this出现在函数中，JavaScript中的this并非函数定义时绑定（箭头函数除外），而是函数执行过程中调用位置绑定

## 四大绑定规则
### 默认绑定

独立函数的调用，可以把这条规则看作是无法应用其他规则时的默认规则，下面的例子使用默认绑定规则，foo()函数没有任何引用进行调用，全局位置直接调用，非严格模式this绑定window，严格模式绑定undefined

```js
functioc foo() {
	console.log(this.a)
}
var a = 2
foo()
```

### 隐式绑定

> 另一条需要考虑的规则是调用位置是否有上下文对象，或者说是否被某个对象拥有或者包含，不过这种说法可能会造成一些误导。

下面的例子中foo函数定义在全局，然后通过引用属性添加到了obj对象中，这里foo函数也是个引用数据类型，这里算不上obj包含foo函数，foo函数独立存在的只是被obj对象的属性引用了，接下来foo函数调用位置使用了obj上下文引用，当函数引用有上下文对象时，隐式绑定规则会把函数调用中的 this 绑定到这个上下文对象
```js
function foo() { 
  console.log( this.a ); 
}
var obj = { 
  a: 2,
  foo: foo 
};
obj.foo(); // 2
```

下面的例子隐式绑定的this会丢失，obj.foo的引用重新赋值给变量bar了，也就是foo函数地址值赋值给变量bar，bar=foo，下面bar()的调用就是调用foo函数，调用位置没引用上下文，因此应用了默认绑定
```js
function foo() { 
	console.log( this.a ); 
}
var obj = { 
  a: 2, 
  foo: foo 
};
var bar = obj.foo; // 函数别名！
var a = "oops, global"; // a 是全局对象的属性 bar(); // "oops, global"
bar(); // "oops, global"

//这样定义的回调函数也会隐式绑定的this丢失，obj.foo通过参数方式专递给setTimeout内部，setTimeout内部会等时间到了调用foo函数，这时的调用的位置也没引用上下文
setTimeout( obj.foo, 100 );
```

### 显示绑定
上面分析隐式绑定时，通过对象包含这个函数（对象属性引用这个函数），间接（隐式）的把函数的this绑定到这个对象上。下面要介绍的就是直接显示的绑定this，下面的例子是通过foo.call(),调用 foo 时强制把它的this绑定到obj，JavaScript 中的“所有”函数都有一些有用的特性，都拥有 call(..) 和 apply(..) 方法，这两个方法的工作原理就是第一个参数是一个对象，用做函数this的绑定对象，第二个参数是需要传入当前函数的参数。

*显式绑定仍然无法解决我们之前提出的丢失绑定问题*

```js
function foo() { 
	console.log( this.a ); 
}
var obj = { 
	a:2 
};
foo.call( obj ); // 2
```
通过硬绑定的方式解决丢失绑定问题
> 我们来看看这个变种到底是怎样工作的。我们创建了函数 bar()，并在它的内部手动调用 了 foo.call(obj)，因此强制把 foo 的 this 绑定到了 obj。无论之后如何调用函数 bar，它 总会手动在 obj 上调用 foo。这种绑定是一种显式的强制绑定，因此我们称之为硬绑定。
```js
function foo() { 
  console.log( this.a ); 
}
var obj = { 
  a:2 
};
var bar = function() { 
  foo.call( obj ); 
};
bar(); 
// 2 setTimeout( bar, 100 ); // 2 // 硬绑定的 bar 不可能再修改它的 this bar.call( window ); //
````
硬绑定是一种非常常用的模式，所以在 ES5 中提供了内置的方法 Function.prototype.bind，下面是bind方法的简单实现，这里的bind函数的实现用到了函数的柯里化，bind函数参数接收一个函数和一个对象，返回一个匿名函数，匿名函数内部在通过apply(...)显示绑定fn的this。

简单点理解就是我给你一个函数和一个对象，你帮我做个this绑定，并且返回一个新函数给我，我在外层调用这个函数时可以传入参数，这个新函数内部在去调用我传入的函数把参数传过去

```js
function foo(something) {
  console.log(this.a, something)
  return this.a + something;
}

function bind(fn, obj) {
  return function() {
      return fn.apply( obj, arguments)
  }
}

var obj = {
  a:2
}

var bar = bind( foo, obj )

var b = bar( 3 ) // 2 3
console.log( b ); // 5

````

### new绑定
在开始之前我们先聊聊JavaScript中new关键字做了那些事，通常new关键字后面都会跟个函数调用（构造函数），构造函数只是一些使用 new 关键字时被调用的函数，

使用 new 来调用函数，或者说发生构造函数调用时，会自动执行下面的操作。

1. 创建（或者说构造）一个全新的对象
2. 新的对象的__proto__属性指向构造函数的prototype属性（原型连接的操作）
3. 新的对象绑定到函数调用的this
4. 如果函数没有返回其他对象，那么 new 表达式中的函数调用会自动返回这个新对象。（如果return普通数据类型返回新创建的对象，如果return自定义对象返回你自定义的对象）

下面例子中是new绑定规则
> 使用 new 来调用 foo(..) 时，我们会构造一个新对象并把它绑定到 foo(..) 调用中的 this 上。new 是最后一种可以影响函数调用时 this 绑定行为的方法，我们称之为 new 绑定。

```js
function foo(a) {
  this.a = a; 
}
var bar = new foo(2); 
console.log( bar.a ); // 2
```

## 优先级

> 现在我们已经了解了函数调用中 this 绑定的四条规则，你需要做的就是找到函数的调用位置并判断应当应用哪条规则。但是，如果某个调用位置可以应用多条规则该怎么办？为了 解决这个问题就必须给这些规则设定优先级，这就是我们接下来要介绍的内容。

1. 默认绑定的优先级是四条规则中最低的
2. 隐式绑定 vs 显式绑定
* 下面代码得出显式绑定优先级高

```js
function foo() { 
  console.log( this.a ); 
}
var obj1 = { 
  a: 2, 
  foo: foo 
};
var obj2 = { 
  a: 3, 
  foo: foo
};

obj1.foo(); // 2
obj2.foo(); // 3 

obj1.foo.call( obj2 ); // 3 显式绑定优先级高
obj2.foo.call( obj1 ); // 2
```

3. new绑定 vs 隐式绑定

* 下面代码得出new绑定比隐式绑定优先级高

```js
function foo(something) {
  this.a = something;
}
var obj1 = { foo: foo };
var obj2 = {}; 

// 隐式绑定
obj1.foo( 2 ); 
console.log( obj1.a ); // 2 

//显示绑定
obj1.foo.call( obj2, 3 ); 
console.log( obj2.a ); // 3

// new绑定
var bar = new obj1.foo( 4 ); 
console.log( obj1.a ); // 2 
console.log( bar.a ); // 4
```

4. new绑定 vs 显示绑定

* new 和 call/apply 无法一起使用，因此无法通过 new foo.call(obj1) 来直接进行测试。
* 可以使用硬绑定来测试它俩的优先级（硬绑定绑定也是显示绑定的一种）
* 下面代码看上去new绑定优先级高于显示绑定
* 详情情况可以看MDN提供的一种 bind(..) 实现

```js
function foo(something) {
  this.a = something; 
}

var obj1 = {};

// 硬绑定
var bar = foo.bind( obj1 ); 
bar( 2 ); 
console.log( obj1.a ); // 2

// new绑定
var baz = new bar(3); 
console.log( obj1.a ); // 2 
console.log( baz.a ); // 3
```

**优先级顺序大致是下面这样**

new绑定 > 显式绑定 > 隐式绑定 > 默认绑定

> 1. 函数是否在 new 中调用（new 绑定）？如果是的话 this 绑定的是新创建的对象。 var bar = new foo() 
> 2. 函数是否通过 call、apply（显式绑定）或者硬绑定调用？如果是的话，this 绑定的是指定的对象。 var bar = foo.call(obj2) 
> 3. 函数是否在某个上下文对象中调用（隐式绑定）？如果是的话，this 绑定的是那个上下文对象。 var bar = obj1.foo() 
> 4. 如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到 undefined，否则绑定到 全局对象。 var bar = foo()

## 箭头函数的this

先引用es20115（es6）标准中对箭头函数的描述

[ES2015原文](http://www.ecma-international.org/ecma-262/6.0/index.html)
> 8.1.1.3 Function Environment Records 

> [[thisBindingStatus]] 

> If the value is "lexical", this is an ArrowFunction and does not have a local this value. 

**大概意思是：箭头函数的 this 绑定状态为 lexical （词法） ，它不创建局部 this 对象。因此它的this标识符查找是在词法作用域中上追溯查找。**

* 也就是箭头函数的this绑定不遵循上面提到的四大绑定原则
* 箭头函数通过词法作用域寻找离自己最近的非箭头父级函数的this
* 箭头函数的绑定无法被修改 ，比如call或appyl无法从新定义箭头函数的this
* 父函数this被修改后箭头函数this也会跟着修改

* 引用《JavaScript高级程序设计》中
> 每个函数在被调用时都会自动取得两个特殊变量：this 和 arguments。

显然箭头函数自身不创建this

对比下面两段代码，效果一样，第一段代码是在ES6之前的写法，t函数是个闭包，t函数自身没有标识符（变量）self，通过词法作用域（作用域链）获取外层foo函数的变量self，self变量缓存了foo函数this。第二段箭头函数的代码也类似，箭头函数自身不创建this关键字，没有就上追溯查找


```js
function foo() {
  var self = this; // lexical capture of this 
  setTimeout(function t(){ 
    console.log( self.a ); 
  }, 100 ); 
}
var obj = { a: 2 };
foo.call( obj ); // 2
```

```js
function foo() { 
  setTimeout(() => { 
    // 这里的 this 在此法上继承自 foo() 
    console.log( this.a ); 
  },100); 
}

var obj = { a:2 };
foo.call( obj ); // 2
```

**总言之箭头函数调用时，this的绑定通过词法作用域查找外层函数的this标识符**

## 总结

* 普通的function函数this绑定遵循四大绑定规则，ES6的箭头函数this绑定遵循词法作用域向上追溯查找

* 用个不是特别恰当的比喻，普通函数this绑定‘类似’动态作用域，及函数的调用位置确定this的绑定。箭头函数this绑定依据静态（词法）作用域了

*如果文章有哪里描述的不够准确欢迎评论区指正交流*



