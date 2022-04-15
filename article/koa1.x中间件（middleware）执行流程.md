# koa1.x中间件（middleware）执行流程

## use函数干了什么
* 先看看源码，忽略掉容错的判断代码，其实就是做了一个push的动作，把我们自己写的fn函数push到this.middleware中，
```js
app.use = function(fn){
  if (!this.experimental) {
    // es7 async functions are not allowed,
    // so we have to make sure that `fn` is a generator function
    assert(fn && 'GeneratorFunction' == fn.constructor.name, 'app.use() requires a generator function');
  }
  debug('use %s', fn._name || fn.name || '-');
  this.middleware.push(fn);
  return this;
};
```
## middleware是怎样运行的
* 首先koa中application.js是初始执行文件，暴露了一个Application构造函数，一般我们new Koa()时构造函数内部会初始化一些属性。上节讲到的middleware 数组也在构造函数的初始化属性中，下面是源码片段
```js
function Application() {
  if (!(this instanceof Application)) return new Application;
  this.env = process.env.NODE_ENV || 'development';
  this.subdomainOffset = 2;
  this.middleware = [];
  this.proxy = false;
  this.context = Object.create(context);
  this.request = Object.create(request);
  this.response = Object.create(response);
}
```
* 接着就是listen函数，listen是绑定在Application构造函数prototype（原型）上的，下面看看listen做了什么, 创建了node的一个原生server服务器。重点看看this.callback函数内部做的了些什么，因为我们知道客户端有请求进来都会触发http.createServer中的回调函数，也就是这里的this.callback
```js
app.listen = function(){
  debug('listen');
  var server = http.createServer(this.callback());
  return server.listen.apply(server, arguments);
};
```
* callback函数内部做了什么了，看重点部分，this.experimental应该是判断是否支持ES7，重点看下 co.wrap(compose(this.middleware))这句代码做了什么，首先是compose，也就是koa项目依赖的一个包koa-compose，（这里提一下，koa1.x依赖的是koa-compose2.x这个版本）
* 我们看看compose都做了些什么，其实koa-compose中就是这短短的20多行代码，compose返回一个generator函数，传入一个参数next
* 第一步判断next有没有，没有的给个默认值空的generator对象（也就是noop，noop调用后会方法一个generator对象）
* 第二步就是获取middleware数组长度
* 第三步倒序循环middleware数组，依次调用数组里的函数，并且把上一次的middleware（中间件）执行后的generator对象，作为参数传入下一个中间件（也就是next，next记录上一次function* () {}执行后的generator对象）
* 最后一步就是return yield *next，这一步很巧妙的应用的yield 语法的特性，因为是倒序循环，最后返回的就是第一个middleware函数调用后返回的generator对象。
```js
app.callback = function(){
  if (this.experimental) {
    console.error('Experimental ES7 Async Function support is deprecated. Please look into Koa v2 as the middleware signature has changed.')
  }
  //重点
  var fn = this.experimental
    ? compose_es7(this.middleware)
    : co.wrap(compose(this.middleware));
  var self = this;

  if (!this.listeners('error').length) this.on('error', this.onerror);

  return function handleRequest(req, res){
    var ctx = self.createContext(req, res);
    self.handleRequest(ctx, fn);
  }
};
```
```js
function compose(middleware){
  return function *(next){
    if (!next) next = noop();

    var i = middleware.length;

    while (i--) {
      next = middleware[i].call(this, next);
    }

    return yield *next;
  }
}

/**
 * Noop.
 *
 * @api private
 */

function *noop(){}
```

**小结** ：执行次序，compose(this.middleware)返回一个 -> generator函数 -> generator函数内主要做了middleware嵌套处理和嵌套关系，结论是调用compose后我们拿到一个generator函数

## tj大神的co库（核心部分）
* 继续接上一节，co.wrap()拿到的参数是一个generator函数，现在我们进co库的源码里看看

* co.wrap()返回了一个叫createPromise函数，createPromise函数内部调用执行了co函数（co库的核心），并且把刚刚compose返回的那个generator函数调用fn.apply(this, arguments)后传入co函数。
```js
co.wrap = function (fn) {
  createPromise.__generatorFunction__ = fn;
  return createPromise;
  function createPromise() {
    return co.call(this, fn.apply(this, arguments));
  }
};
```
* 接下来重点来了，co函数做了什么操作,co函数传入一个参数，这个参数正是刚刚上面提到的，compose返回的generator函数调用fn.apply(this, arguments)后generator对象（gen），co函数返回一个Promise，主要看下Promise回调函数内部代码，首先做了些容错的判断（可以先忽略掉），接下来就是执行onFulfilled函数，onFulfilled内部主要做了两件事，gen.next(res)就是执行我们middleware的yield 
```js
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);

  // we wrap everything in a promise to avoid promise chaining,
  // which leads to memory leak errors.
  // see https://github.com/tj/co/issues/180
  return new Promise(function(resolve, reject) {
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    onFulfilled();

    /**
     * @param {Mixed} res
     * @return {Promise}
     * @api private
     */

    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      next(ret);
      return null;
    }

    /**
     * @param {Error} err
     * @return {Promise}
     * @api private
     */

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    /**
     * Get the next value in the generator,
     * return a promise.
     *
     * @param {Object} ret
     * @return {Promise}
     * @api private
     */

    function next(ret) {
      if (ret.done) return resolve(ret.value);
      var value = toPromise.call(ctx, ret.value);
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  });
}
```
* next(ret)调用next函数，next函数内部首先做了一个判断如果是最后一个yield就直接返回成功，接着就是调用toPromise函数，我看看toPromise函数做了什么,主要做了些Promise类型的判断，重点是这句 **if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);**
```js
function toPromise(obj) {
  if (!obj) return obj;
  if (isPromise(obj)) return obj;
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
  if (isObject(obj)) return objectToPromise.call(this, obj);
  return obj;
}
```
* 如果是generator就在重新执行一遍co函数，这里也就是递归调用了。
* 一般我们middleware函数会这么些
```js
app.use(function* f1(next) {
  console.log('f1: pre next');
  yield next;
});

这里 yield next就是走到co库的
if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj)这段代码来，
在重新走遍co函数的逻辑，也就是执行权交个下个中间件
```
* 最后我们看看这句代码
```js
if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
```
判断如果返回的是一个Promise，就做成功和错误的处理回调，就是为啥传入的中间件函数，yield后面要返回的是一个Promise了，如果成功就重新调用onFulfilled，在执行下个gen.next()，这也就是为啥generator函数可以自动向下执行的原因了，我知道的generator函数是必须的自己手动gen.next()才依次向下继续执行的。
* 上传代码理解下
```js
new Promise(function(resolve, reject) {
  // 我是中间件1
  yield new Promise(function(resolve, reject) {
    // 我是中间件2
    yield new Promise(function(resolve, reject) {
      // 我是中间件3
      yield new Promise(function(resolve, reject) {
        // 我是body
      });
      // 我是中间件3
    });
    // 我是中间件2
  });
  // 我是中间件1
});
```
## 总结
* 其实就是通过generator来暂停函数的执行逻辑来实现等待中间件的效果，通过监听promise来触发继续执行函数逻辑，所谓的回逆也不过就是同步执行了下一个中间件罢了。
**第一个中间件代码执行一半停在这了，触发了第二个中间件的执行，第二个中间件执行了一半停在这了，触发了第三个中间件的执行，然后，，，，，，第一个中间件等第二个中间件，第二个中间件等第三个中间件，，，，，，第三个中间件全部执行完毕，第二个中间件继续执行后续代码，第二个中间件代码全部执行完毕，执行第一个中间件后续代码，然后结束**

盗了一张图

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab65c7ca4f624ac6b00d710341c952b1~tplv-k3u1fbpfcp-zoom-1.image)







