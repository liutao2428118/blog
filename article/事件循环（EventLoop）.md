# 事件循环（EventLoop）

##  浏览器是多进程模型

浏览器是一个多进程模型

* 每个标签页都是一个单独的进程
* 浏览器也有一个主进程，整个用户界面窗口就是主进程
* 渲染进程（浏览器内核），每一个标签页都有一个渲染进程，同一域名下的网址可能共享一个渲染进程，
* 网络进程，处理请求
* 浏览器的第三方插件也是单独的进程

## 渲染进程
渲染进程包含多个线程

* GUI 渲染线程，主要负责渲染页面
* js 引擎线程，负责js的解析与执行
* 事件循环（EventLoop）线程，负责异步代码的调度工作
* DOM 事件，定时器 setTimeout，setInterval，ajax，都是独立的线程

## js 是单线程
毋庸置疑 js 是单线程的，通常我们所说的单线程指的是 js 的主线程（上节提到的 js 引擎线程），js 引擎线程与渲染线程是互斥的，也就是执行 js 代码时，渲染会被挂起，渲染 DOM 时 js 代码不执行。js是单线程，但是渲染进程中是多线程的，在代码中遇到了像定时器 setTimeout，DOM 事件，http 请求等任务时会转交给其他线程（上节提到的线程）去处理的。执行完毕后回调函数会添加到任务队列，这里的执行完毕指的是比如定时器时间到了 ajax 请求成功了，对应的回调函数就会添加到任务队列中。

## 同步与异步
从直观来看 js 的代码是从上至下执行，有因为 js 是单线程同一时间只能做一件事，如果碰到像 ajax 这样从网络中请求数据，如果 ajax 不是异步的可以想象代码会阻塞在此直至请求结果返回，为了解决 js 单线程带来的缺陷，防止阻塞比较好的办法就是排队，异步可以理解成就是在排队，js 最初的设计决定了同一时间干不了多件事，那就只能让耗时较长的任务去排队，等你有结果了并且主线程空闲了我在来执行你。

## 宏任务与微任务

任务队列分宏任务队列和微任务队列，在代码执行过程中，不同的任务类型添加到不同的队列中，并且任务队列是先进先出原则。

**宏任务**

宏任务是宿主环境本身提供的异步方法

* script 脚本
* setTimeout
* setInterval
* ajax
* DOM 事件

**微任务**

微任务是语言标准所提供的，如果在node中process.nextTick是微任务

* Promise.then
* MutationObserver（Mutation Observer API 用来监视 DOM 变动）

## 事件循环

在 EventLoop 中每一次循环被称为 tick，每次 tick 的步骤如下：

1. js 引擎线程中的执行栈首先会执行 script （宏任务），执行完所有的同步代码。
2. 在整体的 script 脚本的执行过程中一定会有同步代码和异步代码，异步代码会根据不同的任务类型，相应的添加到宏任务队列或微任务队列中。
3. 当前 tick 中宏任务执行完毕后，会检查微任务队列，并清空微任务队列执行完所有的微任务
4. 微任务队列清空后，如果宿主环境为浏览器可能会有渲染 DOM 的操作，不过浏览器也会有相应的优化多个 tick 后合并成一次渲染操作。到此当前 tick 结束。
5. 接着就是进入下一次 tick，如果当前执行栈是空闲状态会从宏任务队列中取出一个任务执行，宏任务的执行完毕后紧跟着的又是清空微任务队列，这里可以理解为微任务始终是在当前 tick 末尾执行。
6. 以上步骤循环反复就形成了 EventLoop。

*DOM 的渲染也是事件循环（EventLoop）之中的一个步骤*

上张图加深下理解

![eventLoop](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9ddd94c328549b58ee25080a000620c~tplv-k3u1fbpfcp-zoom-1.image)

## 示例说明
**v1**
```js
<script>
    Promise.resolve().then(() => {
        console.log('Promise1')
        setTimeout(() => {
            console.log('setTimeout2')
        }, 0);
    })
    setTimeout(() => {
        console.log('setTimeout1');
        Promise.resolve().then(() => {
            console.log('Promise2')
        })
    }, 0);
</script>
```

* script 整体会被看做成一个宏任务，进入执行栈，执行完所有同步代码
* 代码中会包含异步代码，对应回调函数添加到相应的队列中
* 外层的 Promise 的 then 方法是微任务，Promise1 回调添加到微任务队列， 外层的 setTimeout 是宏任务，setTimeout1 回调添加到宏任务队列
* script（宏任务）执行完后，清空微任务队列，执行 Promise1 回调执行过程中发现有个setTimeout，setTimeout2 回调添加到宏任务队列
* 宏任务队列中取出一个宏任务执行，也就是执行 setTimeout1 回调，执行过程中发现有个 Promise，Promise2 回调添加到微任务队列
* 再次清空微任务队列，执行 Promise2 回调
* 接着下次 tick 从宏任务队列中取出一个宏任务执行 setTimeout2 回调

依次打印： Promise1，setTimeout1，Promise2，setTimeout2

**v2**
```js
<script>
    Promise.resolve().then(function F1() {
        console.log('promise1')
        Promise.resolve().then(function F4() {
            console.log('promise2');
            Promise.resolve().then(function F5() {
                console.log('promise4');
            }).then(function F6() {
                console.log('promise?');
            })
        }).then(function F7() {
            console.log('promise5');
        })
    }).then(function F2() {
        console.log('promise3');
    }).then(function F3() {
        console.log('promise6');
    })

    
    
     1.首先这段代码放一个script脚本中就是一个宏任务，会首先执行这个宏任务。
     2.然后发现有微任务代码也就是最外层的（Promise.resolve().then）方法。
     3.会把最外层的then方法的回调（F1）添加到微任务队列。
     4.前面的宏任务执行完了（也就是整个script脚本），会马上清空微任务队列，清空也就是依次执行任务队列的回调
     5.执行F1打印promise1，然后F1中有一个微任务F4，F4添加到微任务队列，这时候F1执行完毕，会调用外层的第二个then方法，F2添加到微任务队列。
     6.再次清空微任务队列，依次执行F4，F2。 F4中又发现个微任务F5, F5添加到微任务队列，F4执行完毕后，会执行内层的then方法，F7添加到队列。F2执行完毕后，会调用外层的第三个then，F3添加到微任务队列
     7.再次清空微任务队列，依次执行 F5, F7, F3。 F5执行完毕后，会执行最里层的then，F6添加到微任务队列
     8.再次清空微任务队列 执行 F6 

    任务队列是先进先出的，和执行上下文栈不同
    数组模拟微任务队列
        [F1] 
        [F4, F2]
        [F5, F7, F3]
        [F6]

        // 以次打印 promise1 promise2 promise3  promise4  promise5 promise6 promise?

    
    

</script>

```



