# let与const

## 提升

开始之前我们先聊下js中的提升

```js
 console.log(tmp)
 var tmp = 1 // undefined
```
为什么会打印undefined，我们来分析下，下面引用《你不知道的JavaScript上劵》一段

> 只有声明本身会被提升，而赋值或其他运行逻辑会留在原地。如果提升改变 了代码执行的顺序，会造成非常严重的破坏。

也就是你看到并不是你看到的那样，声明（=号左边）被提升到了最上面， 变量被声明时会初始化赋值undefined

```js
var tmp 
console.log(tmp) // undefined
tmp = 1 
```
函数中变量也是一样的， 声明会被提升到当前函数作用域的最上面
```js
// 你看到的
 function test() {
    console.log(tmp)  // undefined
    var tmp = 2
 }
 // 引擎看到
function test() {
  var tmp
  console.log(tmp)  // undefined
  tmp = 2
}
````

## let

### 没有提升
let声明变量，不存在提升
```js
console.log(a); // 报错ReferenceError
let a = 2;
```

### 临时死区
临时死区(Temporal Dead Zone)，简写为 TDZ

> * let声明的变量不会被提升到作用域顶部，如果在声明之前访问这些变量，会导致报错
> * 这是因为 JavaScript 引擎在扫描代码发现变量声明时，要么将它们提升到作用域顶部(遇到 var 声明)，要么将声明放在 TDZ 中(遇到 let 和 const 声明)。
> * 访问 TDZ 中的变量会触发运行时错误。只有执行过变量声明语句后，变量才会从 TDZ 中移出，然后方可访问。

### 不允许重复声明变量

let不允许在相同作用域内，重复声明同一个变量

```js
function func(arg) {
  let arg; // 报错
}
```

### let全局变量不会挂在顶层对象下面

var声明的全局变量会挂在顶层对象下面，而let不会挂在顶层对象下面
```js
var a = 1;
// 如果在 Node环境，可以写成 global.a
// 或者采用通用方法，写成 this.a
window.a // 1

let b = 1;
window.b // undefined
```

### 块级作用域
es6之前JavaScript中只有全局作用域和函数作用域，let声明变量可以实现块级作用域，“{}”之间就是一个块作用域

```js
function test() {
    for(var i = 0; i <= 10; i++) {
        
    }
    console.log(i)  //11, i是当前函数作用域下面的
}


function test1() {
    for(let i = 0; i <= 10; i++) {
        
    }
    console.log(i)  //报错ReferenceError
}

```

## const

* const基本和let一样的特性，不过const声明的变量不能被修改（也就是一个常量）针对值，如果是对象或者数组内的属性是可以修改，基本数据类型和引用数据类型（地址值）的值不能被修改
* 一旦声明，必须马上赋值
```js
let p; var p1; // 不报错
const p3 = '马上赋值'
const p3; // 报错 没有赋值
```




