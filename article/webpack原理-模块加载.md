# webpack 原理 - 模块加载

* webpack 中的模块加载分同步和异步
    * 异步加载的核心原理是通过JSONP
* webpack 模块加载可以 `commonjs module` 与 `es module` 混用
    * 代码中导入与导出不管用什么方式最后打包编译后都是以 `commonjs module` 方式
    
以下打包编译后的代码示例，源码长这样：
```js
// src/index.js
let title = require('./src/title.js');
console.log(title);
```

```js
// src/title.js
module.exports = 'title';
```


## 基础款

在代码中简单的通过 `commonjs module` 方式导入和导出模块，webpack 打包后简写如下：

``` js
(()=>{
  var modules = {
    './src/title.js':(module,exports,require)=>{
      module.exports = 'title';
    }
  }
  var cache = {};
  function require(moduleId){
    if(cache[moduleId]){//先看缓存里有没有已经缓存的模块对象
      return cache[moduleId].exports;//如果有就直接返回
    }
    //module.exports默认值 就是一个空对象
    var module = {exports:{}};
    cache[moduleId]= module;
    //会在模块的代码执行时候给module.exports赋值
    modules[moduleId].call(module.exports,module,module.exports,require);
    return module.exports;
  }
  //./src/index.js 的代码
  (()=>{
    let title = require('./src/title.js');
    console.log(title);
  })();
})();
```
创建一个 modules 对象存放需要加载的模块，定义 require 函数传入模块ID，通过模块ID在 modules 对象中寻找对应的模块函数，call 的方式调用，传入 module 与 module.exports 绑定当前模块的返回值。接下来执行 index.js 中的代码调用 require 函数。

## common-load-es

在基础款的是基础上打包后的代码增加了两个函数 `require.d ` 和 `require.r`, r 函数主要是给 exports 对象添加 `Symbol.toStringTag` 属性，exports 设置`[object Module]` 类型。并给 exports 还添加了个 __esModule 属性设置成 true。

d 函数主要把 es module 导出方式 `export default` `export xxx ` 转换成 commonjs 的方式  `module.exports.default` `module.exports.xxx`

```js
(()=>{
  var modules = {
    //总结一下，如果原来是es module 如何变成commonjs
    //export default会变成exports.default
    //export xx exports.xx
    './src/title.js':(module,exports,require)=>{
      //不管是commonjs还是es module最后都编译成common.js,如果原来是es module的话，
      /*exports传给r方法处理一下，exports.__esModule=true ，
      以后就可以通过这个属性来判断原来是不是一个es module*/
      require.r(exports);
      require.d(exports,{
        default:()=>DEFAULT_EXPORT,
        age:()=>age
      });
      const DEFAULT_EXPORT = 'title_name';
      const age = 'title_age';
    }
  }

  function require(moduleId){...}
  
  require.d = (exports,definition)=>{
    for(let key in definition){
      //exports[key]=definition[key]();
      Object.defineProperty(exports,key,{enumerable:true,get:definition[key]});
    }
  }
  require.r = (exports)=>{
    //console.log(Object.prototype.toString.call(exports));//[object Module]
    //exports.__esModule=true 
    Object.defineProperty(exports,Symbol.toStringTag,{value:'Module'});
    Object.defineProperty(exports,'__esModule',{value:true});
  }
  //./src/index.js 的代码
  (()=>{
    let title = require('./src/title.js');
    console.log(title);
    console.log(title.age);
  })();
})();
```

## es-load-es

在 common-load-es 基础上打包后的代码把 index.js 变成 modules 对象中的一个待加载模块，并执行了 ` require.r` 函数表示当前 index.js 模块是个 `es module`

```js
(()=>{
  var modules = {
    "./src/index.js":
      ((module,exports,require) => {
        require.r(exports);//先表示这是一个es module
        var title = require("./src/title.js");
        console.log(title.default);
        console.log(title.age);
      }),
    './src/title.js':(module,exports,require)=>{
      require.r(exports);
      require.d(exports,{
        default:()=>DEFAULT_EXPORT,
        age:()=>age
      });
      const DEFAULT_EXPORT = 'title_name';
      const age = 'title_age';
    }
  }

  function require(moduleId){...}
  require.d = (exports,definition)=>{...}
  require.r = (exports)=>{...}
  //./src/index.js 的代码
  (()=>{
    require("./src/index.js");
  })();
})();
```
## es-loda-common
打包后的代码中新增了一个 `require.n` 函数，用于判断导出的方式是 `commonjs module` 还是 `es module`

```js
(()=>{
  var modules = {
    "./src/index.js":
      ((module,exports,require) => {
        require.r(exports);//先表示这是一个es module
        // 引入其实用的还是 require
        var title = require("./src/title.js");
        //n函数的作用，我根本不知道title.js是一个es module还是common
        var title_default = require.n(title);
        console.log((title_default()));//默认值
        console.log(title.age);
      }),
    './src/title.js':(module,exports,require)=>{
      module.exports = {
        name: 'title_name',
        age: 'title_age'
      }
    }
  }
  function require(moduleId){...}
  require.n = (exports)=>{
    var getter = exports.__esModule?()=>exports.default:()=>exports;
    return getter;
  }
  require.d = (exports,definition)=>{...}
  require.r = (exports)=>{...}
  //./src/index.js 的代码
  (()=>{
    require("./src/index.js");
  })();
})();
```

## 异步加载模块

在这之前先聊下 webpack 中的代码块 `chunk` 的概念，如果是通过 `import` 异步加载模块，在打包的过程中就会打包成一个单独的js文件（单独的代码块），也就是不会打包进 main.js 中！

在举个例子如果 A 模块是入口，A 模块中同步引入了 B 和 C 模块，这样打包的时候 A B C 会打包进一个代码块（main.js中），如 A 模块中异步引入了 C 模块, 并且 C 模块中同步引入了 E 模块，这样就会产生两个代码块 main.js 和 c.js，A B C 在同一个代码块， C E 在一个代码块，如果是 C 模块中是异步引 E 模块，这样会产生三个代码块...

以下面源码为例：

```js
// src/index.js
import(/* webpackChunkName: "hello" */'./hello').then(result=>{
      console.log(result.default);
})
```

```js
// src/hello.js
module.exports = 'hello ';
```
打包编译后会生成两个 js 文件，main.js 和 hello.js

打包编译后：

```js
// dist/main.js
(()=>{
  //存放着所有的模块定义，包括懒加载，或者说异步加载过来的模块定义
  var modules = ({});
  var cache = {};
  //因为在require的时候，只会读取modules里面的模块定义
  function require(moduleId){
    if(cache[moduleId]){//先看缓存里有没有已经缓存的模块对象
      return cache[moduleId].exports;//如果有就直接返回
    }
    //module.exports默认值 就是一个空对象
    var module = {exports:{}};
    cache[moduleId]= module;
    //会在模块的代码执行时候给module.exports赋值
    modules[moduleId].call(module.exports,module,module.exports,require);
    return module.exports;
  }
  require.f={};
  //如何异步加载额外的代码块 chunkId=hello
  //2.创建promise，发起jsonp请求
  require.e =(chunkId)=>{
    let promises = [];
    require.f.j(chunkId,promises);
    return Promise.all(promises);//等这个promise数组里的promise都成功之后
  }
  require.p='';//publicPath 资源访问路径
  require.u = (chunkId)=>{//参数是代码块的名字，返回值是这个代码的文件名
    return chunkId+'.main.js';
  }
  //已经安装的代码块 main代码块的名字：0表示已经就绪
  let installedChunks = {
    main:0,
    hello:0
  }
  //3.通过jsonp异步加载chunkId,也就是hello这个代码块
  require.f.j = (chunkId,promises)=>{
    //创建一个新的promise,放到了数组中去
   let promise = new Promise((resolve,reject)=>{
    installedChunks[chunkId]=[resolve,reject];
   });
   promises.push(promise);
   var url = require.p+require.u(chunkId);// /hello.main.js
   require.l(url);
  }
  //http://127.0.0.1:8082/hello.main.js
  //4.通过JSONP请求这个新的url地址
  require.l = (url)=>{
     let script = document.createElement('script');
     script.src = url;
     document.head.appendChild(script);// 一旦添加head里,浏览器会立刻发出请求
  }
  //6.开始执行回调
  var webpackJsonpCallback = ([chunkIds,moreModules])=>{
    //chunkIds=['hello']=>[resolve,reject]
    //let resolves = chunkIds.map(chunkId=>installedChunks[chunkId][0]);
    let resolves = [];
    for(let i=0;i<chunkIds.length;i++){
      let chunkData = installedChunks[chunkIds[i]];
      installedChunks[chunkIds[i]]=0;
      resolves.push(chunkData[0]);
    }
   
    //把异步加载回来的额外的代码块合并到总的模块定义对象modules上去
    for(let moduleId in moreModules){
      modules[moduleId]= moreModules[moduleId];
    }
    resolves.forEach(resolve=>resolve());
  }
  require.d = (exports,definition)=>{
    for(let key in definition){
      Object.defineProperty(exports,key,{enumerable:true,get:definition[key]});
    }
  }
  require.r = (exports)=>{
    Object.defineProperty(exports,Symbol.toStringTag,{value:'Module'});
    Object.defineProperty(exports,'__esModule',{value:true});
  }
  //0.把空数组赋给了window["webpack5"],然后重写的window["webpack5"].push
  var chunkLoadingGlobal = window["webpack5"]=[];
  //然后重写的window["webpack5"].push=webpackJsonpCallback
  chunkLoadingGlobal.push = webpackJsonpCallback;
  //异步加载hello代码块，然后把hello代码块里的模块定义合并到主模块定义里去
  //再去加载这个hello.js这个模块，拿 到模 块的导出结果
  //1.准备加载异步代码块hello
  require.e("hello").then(require.bind(require, "./src/hello.js")).then(result => {
    console.log(result.default);
  })
})();
```

```js
// dist/hello.js
//5.执行window["webpack5"]上的push方法,传递参数[chunkIds,moreModules]
(window["webpack5"] = window["webpack5"] || []).push([["hello"], {
    "./src/hello.js":
        ((module, exports, require) => {
            require.r(exports);
            require.d(exports, {
                "default": () => __WEBPACK_DEFAULT_EXPORT__
            });
            const __WEBPACK_DEFAULT_EXPORT__ = ('hello ');
        })
}]);
```

相比之前编译后的代码会生成：
*  ` require.f` 对象
* ` require.e ` 函数
* `require.p` 属性
* `require.u ` 函数
* `require.f.j` 函数
* `require.l` 函数

并且在 window 上挂了一个属性 `'webpack5' = []` ，并给这个数组重写了 push 方法。

> 这里一定要理解代码块和模块的区别

先看编译后的 hello.js，执行了一个 push 方法传入二维数组，传入代码块ID和和所有的模块，这里 push 的调用是调用的了 main.js 中重写后的 push 方法。

main.js 中是首先执行 `require.e("hello")`, e 方法的主要工作是返回一个 Promise 数组，并且调用 j 方法去实现 jsonp 异步请求代码块资源，jsonp 的本质就是创建一个 script 标签完成远程请求。j 方法另外还创建每个代码块对应的 Promise 对象，并且缓存下了 resolve,reject 方法，做这一步是因为网络请求资源是异步的，不能确定资源何时能返回，等到资源全部返回成功后在调用 resolve 方法执行后续的获取模块的操作。

需要注意的时候 jsonp 的方式请求到对应的代码块资源后会执行里面的代码，也是执行了上面提到的 push 方法，push 是重写后的 webpackJsonpCallback 方法，webpackJsonpCallback 方法内部将获取到的模块添加到 modules 对象中，并依次执行了缓存下来的 resolve 方法，触发了 `require.e("hello").then` 中回调的执行，也就是执行了 require 方法