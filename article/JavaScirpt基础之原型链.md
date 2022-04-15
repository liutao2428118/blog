# JavaScirpt基础之原型链

写在前面，JavaScirpt 是通过原型链来实现面向对象语言中的继承的，按照“你不知道的 JavaScirpt ”中的说法跟准确点，JavaScirpt没有继承只有委托，JavaScript 只是在两个对象之间创建一个关联。真正的继承是子类复制父类的操作。

## 实践中理解

先从一个面试题开始

### 一个面试题开始

```js
Function.prototype.a = () => {
	console.log(1);
}
Object.prototype.b = () => {
	console.log(2);
}
function A() {}
const a = new A();

a.a(); //报错
a.b();  // 2
A.a();  // 1
A.b(); // 2
```

*如果是已经理解了原型和原型链的一眼就可以看出来结果*

### 原型链关系图
![原型链关系图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a031041d46e497dbe675d67730a3d73~tplv-k3u1fbpfcp-zoom-1.image)

### 看图说话

按照原型链关系图分析上面的面试题

* Function.prototype原型对象上添加了一个属性a函数
* Object.prototype原型对象上添加了一个属性b函数
* 自定义一个构造函数A，JavaScirpt中定义一个函数默认会生成一个prototype属性
* new关键字调用构造函数A，创建一个新对象，构造函数A.prototype属性赋值给新对象的__proto__属性
* 调用a对象上的a方法，a对象自身没有，通过__proto__属性找到它的原型对象A.prototype，原型对象也是一个对象，是对象就有原型。原型对象也没有继续向上找，找到原型对象的原型Object.prototype，找到Object.prototype也算是找到头了，如果没有返回undefined，这里用方法的形式调用undefined肯定报错啦
* 调用a对象的b方法，按上面找寻的方式，对象最终都会找到Object.prototype，就会打印2
* 调用A函数对象的a方法，JavaScirpt中所有函数式声明和函数表达式都是Function构造函数的实例，函数其实也是个对象，按照图上的关系这里的a方法会找到Function.prototype上定义的a，打印1
* 调用A函数对象的b方法，按照图上的关系，会先找到Function.prototype上，Function.prototype其实也就个对象字面量，对象都是通过Object构造函数创建的，那可不最后就找到Object.prototype上了，打印2


## 一个个的介绍

### prototype

刚刚上章提到了函数定义后会自动帮我们生成一个prototype属性，那prototype指向哪里，指向一个字面量方式创建的对象，这个对象也是new关键字调用构造函数而创建**实例**的原型

### \__proto__

\__proto__属性不是ES规范中定义的，大部分浏览器默认支持的，方便对象直接访问自己的原型

```js
console.log(a.__proto__ === A.prototype) //true
```

### constructor
constructor是原型对象生成的一个属性，默认指向关联的构造函数，通常对象直接获取constructor，都是它原型上的constructor属性

```js
console.log(a.constructor) // A构造函数
```

### 原型链

什么是原型链，每个对象在创建的时候就会关联上一个原型对象，原型对象也是对象，继续关联它自己的原型对象。每一个对象都会从原型"继承"（委托更贴切）属性，这就是原型链

原型链的头是Object.prototype，找到这就不会继续找下去了，Object.prototype的原型指向的是null，也就是啥也没有了。

向toString，valueOf这些方法就是定义在Object.prototype上的，我们创建的普通对象调用这些方法就通过原型链找到的。

在来捋捋原型链关系图

* p对象是Pers构造函数的实例，p对象的原型指向Pers.prototype
* Pers构造函数是Function构造函数（内置）的实例，Pers函数对象原型指向Function.prototype
* Pers.prototype和Function.prototype原型也是个对象，JavaScirpt中的普通对象都是Object构造函数的实例，最终所有的对象的原型都指向Object.prototype
* Object构造函数（内置）有是Function构造函数（内置）的实例

*是不是很有意思的一个关系网*
