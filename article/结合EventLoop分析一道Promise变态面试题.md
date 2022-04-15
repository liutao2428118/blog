# 结合EventLoop分析一道Promise变态面试题

## 题目

如果对EventLoop原理不是很了解可以看我另外一篇文章 [事件循环](https://juejin.cn/post/6898975636035993607)

```js
  let a;
  const b = new Promise((resolve, reject) => {
      console.log('promise1');
      resolve();
  }).then(function b1() {
      console.log('promise2');
  }).then(function b2() {
      console.log('promise3');
  }).then(function b3() {
      console.log('promise4');
  });

  a = new Promise(async (resolve, reject) => {
      console.log(a);
      await b;
      console.log(a);
      console.log('after1');
      await a
      resolve(true);
      console.log('after2');
  });

  console.log('end');
```

**先上结果：** promise1，undefined， end， promise2， promise3， promise4， Promise {pending}， after1

## 逐步分析

1.这段代码放到script中，整体上就是个宏任务，首先执行这个宏任务，代码从上到下执行

2.第一个new Promise()是调用Promise构造函数，构造函数内部会马上执行你传入的回调(resolve, reject) =>{...}，不用怀疑首先打印‘promise1’，接着执行了resolve()。（这块回调中的代码可以看做是同步代码）

3.上面resolve()的执行会返回Promise {fulfilled}，下面的第一个then会被调用，b1回调不会马上执行，then方法属于微任务，所以b1回调会被添加到微任务队列

```js
微任务队列: [b1]
```

4.下面的两个then都不会执行，因为要等b1回调执行完，返回新的Promise {fulfilled}

5.进入第二个new Promise()调用Promise构造函数，执行回调async (resolve, reject) =>{...}，这一块同样看做是同步的。
打印console.log(a)的结果‘undefined’，这块为啥是undefined了我觉得是调用Promise构造函数同时马上打印a变量，Promise对象还没创建成功所以还没赋值给a变量

```js
...
a = new Promise(async (resolve, reject) => {
    console.log(a);
    await b;
    console.log(a);
    console.log('after1');
    await a
    resolve(true);
    console.log('after2');
});

console.log(a); // Promise {<pending>} 如果这里打印a赋值是成功了

console.log('end');
```

6.await b不会马上执行，它要等上面b3回调执行后返回新Promise {fulfilled}， await b可以整体看做是下面代码：

```js
// await b在等b3回调执行完返回成功的新Promise对象

...省略的代码
.then(function b3() {
    console.log('promise4');
}).then(function b4() {
    console.log(a); 
    console.log('after1');
})

await b // promise.then(function b4() {console.log(a); console.log('after1');})

```

7.await a永远不会执行了，它在自己等自己

8.执行最后一句同步代码 console.log('end') ，打印'end'

9.整个script执行完后（宏任务），会立刻清空微任务队列
```js
  // 第三步的时候微任务队列添加了b1回调
  
  微任务队列: [b1] 
  
  // 任务队列是先进先出策略
  
  微任务队列: []  //清空
  
  b1回调函数放了执行栈执行
  
  // 打印 promise2
    
```

10. b1回调执行成功后会返回新Promise {fulfilled}，调用下面的then方法，b2回调添加到微任务队列

```js
微任务队列: [b2]
```
11.再次清空微任务队列

```js
微任务队列: [] // 清空

b2回调函数放了执行栈执行

// 打印 promise3

```
12.b2回调执行成功会返回新Promise {fulfilled}，调用下面的then方法，b3回调添加到微任务队列

```js
微任务队列: [b3]
```

13.再次清空微任务队列

```js
微任务队列: [] // 清空

b3回调函数放了执行栈执行

// 打印 promise4

```

14.b3回调执行成功会返回新Promise {fulfilled}，下面就是连接上第六步了, 调用await b返回的then方法，b4添加到微任务队列

```js
//第六步的代码拿过来
await b // promise.then(function b4() {console.log(a); console.log('after1');})

微任务队列: [b4]
```

15.再次清空微任务队列

```js
微任务队列: [] // 清空

b4回调函数放了执行栈执行

// 打印 Promise {<pending>}， after1

```

*前面说了await a是自己等自己，await a永远等不到Promise {fulfilled}，then不调用相应的回调也不会添加到队列，也就没有后续的清空队列，执行回调函数*

```js
  其实长这样：

  await a // promise.then(() => {resolve(true); console.log('after2');})
    
```

* 结合逐步分析在看最开始给出的打印顺序结果，大概也就可以理解原理了

### 聊聊这里的 await

先把上面的题目改改

```js
...我是省略代码
a = new Promise(async (resolve, reject) => {
    console.log(a);
    await console.log(b); //看出和上面有什么不同了吗， Promise.resolve(console.log(b)).then(() => { console.log(a);console.log('after1')})
    console.log(a);
    console.log('after1');
    await a
    resolve(true);
    console.log('after2');
});

console.log(a); // Promise {<pending>} 如果这里打印a赋值是成功了

console.log('end');
```
打印结果变成了

```js
promise1
undefined
Promise {<pending>}
end
promise2
Promise {<pending>}
after1
promise3
promise4
```
分析下

* 这里的await console.log(b)被Promise.resolve包裹了一层，和上面的b3回调执行成功后的返回的Promise对象完全没关系了，这里的Promise.resolve返回一个全新的Promise对象

* 同步代码走到这里的时候，Promise.resolve(console.log(b))中的b会被立即打印出来‘Promise {pending}’

* 同时 Promise.resolve(console.log(b))的then方法的回调() => { console.log(a);console.log('after1')}，会被添加到微任务队列

```js
// 上面我们到第三步时微任务队列是这样的
微任务队列: [b1]

// 现在代码被我们改后,第三步之后微任务队列是下面这样的

微任务队列: [b1， () => { console.log(a);console.log('after1')}]
```

* 后续同步代码执行完，就会清空微任务队列

```js
	
微任务队列: [b1， () => { console.log(a);console.log('after1')}]

微任务队列: [] //清空

// 先进先出原则

b1回调函数放入执行栈执行，
() => { console.log(a);console.log('after1')} 放入执行栈执行
```

* 这就是为啥 'a: Promise {pending}'和'after1'会在promise2之后打印，promise3和promise4之前打印了

```js
() => { console.log(a);console.log('after1')}这个回调是和b1回调是一起先后依次执行的
```

以上是自己的个人分析，不一定全部描述都是正确的，如果有错误或者不同观点，欢迎评论区指出和讨论


