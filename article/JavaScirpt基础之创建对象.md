# JavaScirpt基础之创建对象

直接引用JavaScript高级程序设计里的，因为里面已经写的很好了，这里只是自己的一个梳理笔记

## 字面量与new Object()

```js
var a = {}

var b = new Object()

两种方式等价
```

适合场景：创建属性不相同的单个对象，字面量是new Object()的简写

缺陷：使用同一个接口创建很多对象（属性相同的），会产生大量的重复代码

## 工厂模式

```js
function createPerson(name, age, job) {
  var o = new Object()
  o.name = name
  o.age = age
  o.job = job
  o.sayName = function() {
      alert(this.name)
  }
  return o
}

var person = createPerson('Nicholas', 29, 'Software Engineer')
```

适合场景：重复创建多个相同属性的独立对象，工厂模式解决了创建多个相似对象

缺陷：对象类型无法识别，因为实例全部指向一个原型（Object.prototype）

## 构造函数模式

```js
function Person(name, age, job) {
  this.name = name
  this.age = age
  this.job = job
  this.seyName = function() {
      alert(this.name)
  }
}

var person = new Person('Nicholas', 29, 'Software Engineer')
```

适合场景：与工厂模式很类似创建多个相似对象，不过解决了对象类型识别的问题，alert(person1 instanceof Person) // true

缺陷：每次调用构造函数，实例都会重新创建一遍方法，方法都是每个实例独有的

## 原型模式

```js
function Person(){ 
} 

Person.prototype.name = "Nicholas"
Person.prototype.age = 29
Person.prototype.job = "Software Engineer"
Person.prototype.sayName = function(){ 
 alert(this.name); 
}

var person1 = new Person()
person1.sayName() //"Nicholas" 

var person2 = new Person()
person2.sayName(); //"Nicholas"
```

适合场景：方法不用重复创建了，所有实例共享属性和方法

缺陷：省略了为构造函数传递初始化参数，所有实例共享属性和方法（优点也是缺点）

### 原型模式的简写

```js
function Person(){ 
} 

Person.prototype = { 
 name : "Nicholas", 
 age : 29, 
 job: "Software Engineer", 
 sayName : function () { 
 alert(this.name); 
 } 
}
```
适合场景: 同上，另外封装的好看点了

缺陷： 同上，另外constructor丢失了，如果prototype是在实例化之后被覆盖的，实例访问属性会找不到

## 组合模式，构造函数+原型

```js
function Person(name, age, job){ 
 this.name = name
 this.age = age
 this.job = job 
 this.friends = ["Shelby", "Court"]
} 
Person.prototype = { 
 constructor : Person, 
 sayName : function(){ 
   alert(this.name)
 } 
} 
var person1 = new Person("Nicholas", 29, "Software Engineer");
var person2 = new Person("Greg", 27, "Doctor")
```
适合场景：实例属性各自属于实例本身，方法共享不会重复创建，广泛、认同的模式

缺陷：暂时没看到

## 动态原型模式

```js
function Person(name, age, job) {
	this.name = name
    this.age = age
    this.job
    
    if(typeof this.seyName != 'function') {
    	Person.prototype.seyName = function() {
        	alert(this.name)
        }
    }
}

var person =  new Person("Nicholas", 29, "Software Engineer")
person.seyName()
```

适合场景：那些想把构造函数和原型写到一起的，这里只在 sayName()方法不存在的情况下，才会将它添加到原型中。这段代码只会在初次调用构造函数时才会执行。此后，原型已经完成初始化，不需要再做什么修改了。

缺陷：这里的prototype不能写成字面量形式直接覆盖，在创建实例new的时候构造函数的prototype会赋值给实例的__proto__, new Person()的同时等于调用了Person函数，就会执行if语句，如果这个时候直接字面量覆盖Person.prototype，实例的原型的指向并没更改，调用person.seyName还是会去原来的原型对象上找，找不到肯定就报错了

## 寄生构造函数模式
```js
fubction Person(name, age, job) {
	var o = new Object()
    o.name = name
    o.age = age
    o.job = job
    o.sayName = function() {
 		alert(this.name)   
    }
    
    return o
}

var friend = new Person("Nicholas", 29, "Software Engineer")
friend.sayName() //"Nicholas"
```

适合场景：与工厂模式很类似，只不过用构造函数伪装了下，这里要注意的是使用new关键字如果构造函数没明确return默认是返回新创建的对象，如果明确了 return o 会返回手动创建的对象，如果返回的是基础类型默认还是返回新创建的对象

这个模式可以在特殊的情况下用来为对象创建构造函数。假设我们想创建一个具有额外方法的特殊数组。由于不能直接修改 Array 构造函数，因此可以使用这个模式。

```js
function SpecialArray(){ 
   //创建数组
   var values = new Array(); 
   //添加值
   values.push.apply(values, arguments)
   //添加方法
   values.toPipedString = function(){ 
   		return this.join("|"); 
   }; 
 
 //返回数组
 return values; 
} 
var colors = new SpecialArray("red", "blue", "green");
alert(colors.toPipedString()); //"red|blue|green"
```

缺陷：返回的对象与构造函数或者与构造函数的原型属性之间没有关系；不能依赖 instanceof 操作符来确定对象类型

## 稳妥构造函数模式

```js
function Person(name, age, job) {
  var o = new Object()
  o.sayName = function() {
      alert(name)
  }
  return o
}

var friend = Person("Nicholas", 29, "Software Engineer")
friend.sayName() //"Nicholas"
friend.name = "daisy" // 没用
```
适合场景：稳妥模式与寄生模式不同在于没用this和new关键字，不会创建新对象。没用公共属性，稳妥对象最适合在一些安全的环境中。

缺陷：稳妥构造函数模式也跟工厂模式一样，无法识别对象所属类型。
