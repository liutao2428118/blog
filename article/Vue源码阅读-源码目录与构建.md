# Vue源码阅读-源码目录与构建

Vue.js 的源码利用了 Flow 做了静态类型检查，所以了解 Flow 有助于我们阅读源码。[Flow](https://flow.org/en/docs/getting-started/)

Vue 源码阅读版本 vue2.5.17-beta

## 源码目录结构

目录结构
```
├── scripts ------------------------------- 包含与构建相关的脚本和配置文件
│   ├── alias.js -------------------------- 源码中使用到的模块导入别名
│   ├── config.js ------------------------- 项目的构建配置
├── build --------------------------------- 构建相关的文件，一般情况下我们不需要动
├── dist ---------------------------------- 构建后文件的输出目录
├── examples ------------------------------ 存放一些使用Vue开发的应用案例
├── flow ---------------------------------- JS静态类型检查工具的类型声明
├── package.json
├── test ---------------------------------- 测试文件
├── src ----------------------------------- 源码目录
```

源码都在src目录下
```
src
├── compiler        # 编译相关 
├── core            # 核心代码 
├── platforms       # 不同平台的支持
├── server          # 服务端渲染
├── sfc             # .vue 文件解析
├── shared          # 共享代码
````

主要关注下src目录

### compiler
compiler 目录包含Vue.js的所有编译相关的代码，我们传入的options中定义template，需要compiler来编译成render函数。

编译的工作可以在构建时做（借助 webpack、vue-loader 等辅助插件）；也可以在运行时做，使用包含构建功能的 Vue.js。显然，编译是一项耗性能的工作，所以更推荐前者——离线编译。vue-cli（官方脚手架）就是构建时做编译

### core
core 目录包含了 Vue.js 的核心代码，包括内置组件 全局API封装 Vue实例化 观察者 虚拟DOM 工具函数，这里的代码是 Vue.js 灵魂。

### platforms
Vue 是支持跨平台MVVM框架，能跑在 web 端，也可以配合 weex 跑在 native 客户端上。platform 是 Vue.js 的入口，2 个目录代表 2 个主要入口，分别打包成运行在 web 上和 weex 上的 Vue.js。

### server
Vue 2.0 版本后支持了服务端渲染，相关服务端渲染的代码都在server目录。

### sfc
通常我们都是用 webpack 构建 Vue 项目，这个目录下的代码逻辑会把 .vue 文件内容解析成一个 JavaScript 的对象。

### shared
Vue 中工具方法都定义在这个目录里，共享给其他模块

## Vue 源码的构建

### 构建脚本

Vue 是基于 Rollup 构建的，和我们平时项目都是 webpack 构建不一样。[rollup](https://github.com/rollup/rollup)

从项目目录的 package.json 文件开始

script字段是npm的执行文件，Vue 部分构建命令如下

```
{
  "script": {
    "build": "node scripts/build.js",  // node运行 scripts目下的build.js
    "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer", // 在上面的基础添加了一些环境参数，build.js内部通过process.argv拿到参数
    "build:weex": "npm run build -- weex" //同上
  }
}
```

### 构建过程

进入 scripts/build.js 文件

```js
//scripts/build.js

// 拿到我所要构建的所有配置
let builds = require('./config').getAllBuilds()

// filter builds via command line arg，根据不同的环境配置不同的参数（下面进行的就是过滤参数配置）
if (process.argv[2]) {
const filters = process.argv[2].split(',')
builds = builds.filter(b => {
  return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
})
} else {
// filter out weex builds by default
builds = builds.filter(b => {
  return b.output.file.indexOf('weex') === -1
})
}
// 拿到配置参数以后，通过build函数，进行build配置
build(builds)
```

在看看 builds 从 scripts/config.js 拿到了是些什么配置数据

```js
// scripts/config.js

// aliases 提供到真实文件地址的映射
const aliases = require('./alias')  // scripts/alias.js
const resolve = p => {
  const base = p.split('/')[0] // base 就是 'web'
  if (aliases[base]) { // aliases[base] = web: resolve('src/platforms/web')
    return path.resolve(aliases[base], p.slice(base.length + 1)) // p.slice(base.length + 1) 拿到除base的后半部分。拼接路径
  } else {
    // 不存在aliases里面的，比如dist。就是直接走到整个路径下，找到对应的文件
    return path.resolve(__dirname, '../', p)
  }
}

// builds keys里面分别是不同版本vue.js的配置
/**
 * @entry : 入口
 * @dest ： 目标
 * @format ： 构建出来的不同文件格式的js ，有点像早期的amd，cmd到现在的es等
 * @banner : 局部变量，配置到顶部像自动生成的一些注释
 */
const builds = {
  // Runtime only (CommonJS). Used by bundlers e.g. Webpack & Browserify
  'web-runtime-cjs': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.common.js'),
    format: 'cjs',
    banner
  },
  // Runtime+compiler CommonJS build (CommonJS)
  'web-full-cjs': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.common.js'),
    format: 'cjs',
    alias: { he: './entity-decoder' },
    banner
  },
  // Runtime only (ES Modules). Used by bundlers that support ES Modules,
  // e.g. Rollup & Webpack 2
  'web-runtime-esm': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.esm.js'),
    format: 'es',
    banner
  },
  // Runtime+compiler CommonJS build (ES Modules)
  'web-full-esm': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.esm.js'),
    format: 'es',
    alias: { he: './entity-decoder' },
    banner
  },
  // runtime-only build (Browser)
  'web-runtime-dev': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.js'),
    format: 'umd',
    env: 'development',
    banner
  },
  // runtime-only production build (Browser)
  'web-runtime-prod': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.min.js'),
    format: 'umd',
    env: 'production',
    banner
  },
  // Runtime+compiler development build (Browser)
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.js'),
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
  // Runtime+compiler production build  (Browser)
  'web-full-prod': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.min.js'),
    format: 'umd',
    env: 'production',
    alias: { he: './entity-decoder' },
    banner
  },
  // Web compiler (CommonJS).
  'web-compiler': {
    entry: resolve('web/entry-compiler.js'),
    dest: resolve('packages/vue-template-compiler/build.js'),
    format: 'cjs',
    external: Object.keys(require('../packages/vue-template-compiler/package.json').dependencies)
  },
  // Web compiler (UMD for in-browser use).
  'web-compiler-browser': {
    entry: resolve('web/entry-compiler.js'),
    dest: resolve('packages/vue-template-compiler/browser.js'),
    format: 'umd',
    env: 'development',
    moduleName: 'VueTemplateCompiler',
    plugins: [node(), cjs()]
  },
  
  .......
}


// getAllBuilds: 遍历builds配置，调用函数
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(
    )
}
```
* **cjs** CommonJs Module，遵循CommonJs Module规范的文件输出
* **es** ES Modules，使用ES6的模板语法输出
* **amd** AMD Module,遵循AMD Module规范的文件输出
* **umd** 支持外链规范的文件输出，此文件可以直接使用script标签

### Runtime Only VS Runtime + Compiler

* Runtime Only
如果我们在 Vue 中直接使用 render 函数编写 DOM 结构，或者借助如 webpack 的 vue-loader 工具把 .vue 文件编译成 JavaScript，.vue 中的 template 部分会被 vue-loader 编译成 render 函数（预编译），这时就可以使用 Runtime Only 版本。

* Runtime + Compiler
如果在 Vue 中直接使用 template  属性并传入一个字符串，且也不是用 webpack 构建项目，无法使用 vue-loader 工具做预编译，就必须使用 Runtime + Compiler 版本

```js
// 需要编译器的版本
new Vue({
  template: '<div>{{ hi }}</div>'
})

// 这种情况不需要编译器
new Vue({
  render (h) {
    return h('div', this.hi)
  }
})
```

## 总结

了解了 Vue.js 源码的目录结构，知道了 Vue.js 是通过 Rollup 构建的，构建过程是通过 nodejs 获取不同版本的 entry 与 dest 路径，在通过 Rollup 打包，不同版本 Vue 打包入口和输出完全不同



