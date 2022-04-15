# webpack 原理- loader 的执行

## loader 的执行顺序

默认情况下是：**从右往左，从下往上** 
```js
 {
    test:/\.js$/,
    use:['normal-loader1','normal-loader2']
  }
```
先执行 normal-loader2 在执行 normal-loader1。

不过还有额外的情况，同一资源可以配置多个 rules 并且执行顺序可以自己把控

loader 分为一下几种形式：

* 前置 loader
```js
{
    test:/\.js$/,
    enforce:'pre',//一定先执行
    use:['pre-loader1','pre-loader2']
}
```
* 普通 loader
```js
{
    test:/\.js$/,
    use:['normal-loader1','normal-loader2']
}
```
* 内联 loader
内联 loader 有点特殊，它并不是在 rules 中定义，如果使用了 inline-loader 又不想 pre-loader 和 normal-loader 再执行，可以使用 `-!` 禁止

| 符号 | 结果 |
| --- | --- |
|  `-!` |  禁止 pre-loader 和 normal-loader 再执行 |
|  `!` |  禁止 normal-loader 再执行 |
|  `!!` |  禁止 inline-loader 外的其他 loader 再执行 |

```js
const xxx = require('-!inline-loader1!inline-loader2!./index.js')
```

* 后置 loader
```js
{
    test:/\.js$/,
    enforce:'post',//post webpack保证一定是后执行的
    use:['post-loader1','post-loader2']
}
```

这几种 loader 的执行顺序分别是上面定义的这样 : **pre -> normal -> inline -> post**

实现原理大概是下面这样：

```js
let request = `inline-loader1!inline-loader2!./index.js`
let parts = request.replace(/^-?!+/,'').split('!')
//最后一个元素就是要加载的资源了
let resource = path.resolve(__dirname,'src',parts.pop())
// loaders 可以是你自定义存放 loader 的目录，也可以是 node_modules
let resolveLoader = (loader)=>path.resolve(__dirname,'loaders',loader)
let inlineLoaders = parts.map(resolveLoader)

let rules = options.module.rules
let preLoaders = [];
let postLoaders = [];
let normalLoaders = [];
for(let i=0;i<rules.length;i++){
    let rule = rules[i]
    // 判断文件后缀是否一致
    if(rule.test.test(resource)){
       if(rule.enforce =='pre'){
         preLoaders.push(...rule.use);
       }else if(rule.enforce == 'post'){
        postLoaders.push(...rule.use);
       }else{
        normalLoaders.push(...rule.use);
       }
    }
}
// 拼接上全路径
preLoaders = preLoaders.map(resolveLoader)
postLoaders = postLoaders.map(resolveLoader)
normalLoaders = normalLoaders.map(resolveLoader)
let loaders = [];
if(request.startsWith('!!')){//noPrePostAutoLoaders
    loaders=[...inlineLoaders];
}else if(request.startsWith('-!')){//noPreAutoLoaders
    loaders=[...postLoaders,...inlineLoaders];
}else if(request.startsWith('!')){//不要普通 loader
    loaders=[...postLoaders,...inlineLoaders,...preLoaders];
}else{
   // inline-loader 没有前缀符合 按顺序添加
  loaders=[...postLoaders,...inlineLoaders,...normalLoaders,...preLoaders];
}

runLoaders({
    resource,
    loaders,
    context: {name:'my'},
	readResource: fs.readFile.bind(fs)
}, function(err, result) {
    console.log(err);
    console.log(result.result,result.resourceBuffer.toString());
})

```
* 首先获取资源路径，如果有 inline-loader 切割出来，拼接 loader 的绝对路径，添加到 `lineLoaders` 数组
* 获取配置文件中的 rules ，遍历把相同的 loader 分别添加到到对应的数组，并且拼接绝对路径
* 判断 inline-loader 中是否有前缀符号，并把对应的 loader 剔除，如果没有就按顺序添加到 `loaders` 中
* `loaders` 传入 runLoaders 执行，runLoaders 就是下面章节要讲的了

## loader 的组成
loader 是由两部分组成的 `pitch` 和 `normal`，真意义上来说 loader 其实就是一个函数，`pitch` 是挂在这函数对象的上的另一个函数

以上除了我们提到如何配置 loader 会产生不同的执行顺序，在 loader 内部还有 `pitch` 和 `normal` 执行顺序


![loader-zx.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfe04838923448a79af0b1085e1e944d~tplv-k3u1fbpfcp-watermark.image?)

首先会执行最左边 loader 的 `pitch` 在依次往后执行完所有的 `pitch` , 在加载资源也就是源码，最后在依次从最右边的 loader 的 `normal` 开始执行

loader 的本质就是拿到资源文件处理，处理完后给到下一个 loader 在进行处理，最后把结果返回给 webpack。

pitch 的意义，pitch 在没有返回值或者没定义 pitch 情况下是不起什么作用，如果 pitch 有返回值会拦截之后 loader 的执行，把结果直接返回给上个 loader


pitch 本身是个函数并且接收三个参数，分别是 `remainingRequest` `previousRequest` `data`

* `remainingRequest` 剩下 loader 的加载路径
* `previousRequest` 之前 loader 的加载路径

style-loader 的实现就是通过 pitch

```js
normal.pitch = function(remainingRequest,previousRequest,data){
    let style =  `
     let style = document.createElement('style');
     style.innerHTML = require(${loaderUtils.stringifyRequest(this,"!!"+remainingRequest)}).toString();
     document.head.appendChild(style);
    `;  
    return style;
}
```

一般情况 style-loader 是最左侧的 loader，也是最后执行的 loader，因为 style-loader  右侧的 loader 通常是返回一个 js 代码字符串，style-loader 并不怎么好处理这段 js 字符串，所以在 pitch 中直接拦截掉后续 loader 的执行，通过 webpack require 方式加载剩余 loader 的处理结果，`remainingRequest` 就是剩余要处理的 loader 路径（style-loader 右侧的 loader 都会走一遍），前缀加上符号 `!!` 表示除了 inline-loader 其他 loader 都不用执行了（避免重复的在执行了）


![loader1.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27df285019e744949c8b37624e9901f8~tplv-k3u1fbpfcp-watermark.image?)

## loader-runner
在 webpack 中 loader-runner 是被分割出来的单独的库，所有 loader 的执行都是依赖于 loader-runner ，下面探究下 loader-runner  的实现。

loader-runner 会暴露一个 `runLoaders` 函数
```js
function runLoaders(options,callback){
    let resource = options.resource || '';//要加载的资源 c:/src/index.js?name=zhufeng#top
    let loaders = options.loaders || [];//loader绝对路径的数组
    let loaderContext = options.context||{};//这个是一个对象，它将会成为loader函数执行时候的上下文对象this
    let readResource = options.readResource||readFile;
    let loaderObjects = loaders.map(createLoaderObject);
    loaderContext.resource=resource;
    loaderContext.readResource = readResource;
    loaderContext.loaderIndex = 0;//它是一个指标，就是通过修改它来控制当前在执行哪个loader
    loaderContext.loaders = loaderObjects;//存放着所有的loaders
    loaderContext.callback = null;
    loaderContext.async = null;//它是一个函数，可以把loader的执行从同步改为异步
    Object.defineProperty(loaderContext,'request',{
        get(){
            return loaderContext.loaders.map(l=>l.request).concat(loaderContext.resource).join('!')
        }
    });
    Object.defineProperty(loaderContext,'remainingRequest',{
        get(){
            return loaderContext.loaders.slice(loaderContext.loaderIndex+1).concat(loaderContext.resource).join('!')
        }
    });
    Object.defineProperty(loaderContext,'currentRequest',{
        get(){
            return loaderContext.loaders.slice(loaderContext.loaderIndex).concat(loaderContext.resource).join('!')
        }
    });
    Object.defineProperty(loaderContext,'previousRequest',{
        get(){
            return loaderContext.loaders.slice(0,loaderContext.loaderIndex).join('!')
        }
    });
    Object.defineProperty(loaderContext,'data',{
        get(){
            let loaderObj = loaderContext.loaders[loaderContext.loaderIndex];
            return loaderObj.data;
        }
    });
    let processOptions = {
        resourceBuffer:null
    }
    iteratePitchingLoaders(processOptions,loaderContext,(err,result)=>{
        callback(err,{
            result,
            resourceBuffer:processOptions.resourceBuffer
        });
    });
}
```
把传入的配置对象的属性赋值给 `loaderContext` ，这里比较重要的是 createLoaderObject 函数的实现，字面意思就是创建 Loader 对象, 之后的操作都是围绕着这个 Loader 对象

```js
function createLoaderObject(request){
    let loaderObj = {
        request,
        normal:null,//loader函数本身
        pitch:null,//pitch函数本身
        raw:false,//是否需要转成字符串,默认是转的
        data:{},//每个loader都会有一个自定义的data对象，用来存放一些自定义信息
        pitchExecuted:false,//pitch函数是否已经执行过了
        normalExecuted:false//normal函数是否已经执行过了
    }
    let normal = require(loaderObj.request);
    loaderObj.normal = normal;
    loaderObj.raw = normal.raw;
    let pitch = normal.pitch;
    loaderObj.pitch = pitch;
    return loaderObj;
}
```
接着调用 `iteratePitchingLoaders` 函数
```js
function iteratePitchingLoaders(processOptions,loaderContext,finalCallback){
    if(loaderContext.loaderIndex>=loaderContext.loaders.length){
        return processResource(processOptions,loaderContext,finalCallback);
    }
   let currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];
   if(currentLoaderObject.pitchExecuted){
    loaderContext.loaderIndex++;
    return iteratePitchingLoaders(processOptions,loaderContext,finalCallback);
   }
   let pitchFunction = currentLoaderObject.pitch;
   currentLoaderObject.pitchExecuted =true;//表示pitch函数已经执行过了
   if(!pitchFunction)//如果此loader没有提供pitch方法
     return iteratePitchingLoaders(processOptions,loaderContext,finalCallback);
   runSyncOrAsync(pitchFunction,loaderContext,
    [loaderContext.remainingRequest,loaderContext.previousRequest,loaderContext.data]
    ,(err,...values)=>{
        if(values.length>0 && !!values[0]){
            loaderContext.loaderIndex--;//索引减1，回到上一个loader,执行上一个loader的normal方法
            iterateNormalLoaders(processOptions,loaderContext,values,finalCallback);
        }else{
            iteratePitchingLoaders(processOptions,loaderContext,finalCallback);
        }
   });
}
```
首先判断是否越界，如果是执行到最右侧 loader 的 Pitch 就去加载源码资源，通过 loaderContext.loaderIndex 拿到当前 Loader 对象，并且 pitchExecuted 设置为 true 表示已经执行过 Pitch 了，最后通过 runSyncOrAsync 函数调用 Pitch，如果有返回值就 loaderContext.loaderIndex-- 执行前一个 loader 的 normal ，如果没返回值就递归调用 iteratePitchingLoaders 执行下一个 loader 的 pitch

```js
function runSyncOrAsync(fn,context,args,callback){
  let isSync = true;//是否同步，默认是的
  let isDone = false;//是否fn已经执行完成,默认是false
  const innerCallback = context.callback = function(err,...values){
        isDone= true;
        isSync = false;
        callback(err,...values);
  }
  context.async = function(){
    isSync=false;//把同步标志设置为false,意思就是改为异步
    return innerCallback;
  }
  let result = fn.apply(context,args);
  if(isSync){
    isDone =true;//直接完成
    return callback(null,result);//调用回调
  }
}
```
iterateNormalLoaders 函数
```js
function iterateNormalLoaders(processOptions,loaderContext,args,finalCallback){
    if(loaderContext.loaderIndex<0){// 如果索引已经小于0了，就表示所有的normal执行完成了
        return finalCallback(null,args);
    }
    let currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];
   if(currentLoaderObject.normalExecuted){
    loaderContext.loaderIndex--;
    return iterateNormalLoaders(processOptions,loaderContext,args,finalCallback);
   }
   let normalFunction = currentLoaderObject.normal;
   currentLoaderObject.normalExecuted =true;//表示pitch函数已经执行过了
   convertArgs(args,currentLoaderObject.raw);
   runSyncOrAsync(normalFunction,loaderContext,args,(err,...values)=>{
    if(err)finalCallback(err);
    iterateNormalLoaders(processOptions,loaderContext,values,finalCallback);
   });
}
```
首先还是边界处理，如果所有的 loader 执行完了调用 finalCallback 返回处理后的源码，接着就是 loaderContext.loaderIndex-- 递归调用 iterateNormalLoaders 函数处理完所用的 normal

## 总结

所用 loader 执行是通过 loader-runner 完成的，loader 完整流程是 先从最左边的 loader 的 pitch 开始到最右边 loader 的 pitch，在获取资源，在从最右边 normal 到最左边的 normal