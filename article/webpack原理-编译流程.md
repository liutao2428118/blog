# webpack 原理-编译流程

通常我们使用 webpack 都是通过命令行，其实 webpack 也可是 nodejs 的一个模块引入并调用。命令行也是在这基础上封装了一层。

```js
const webpack = require('webpack');
const options = require('./webpack.config');
const compiler = webpack(options);

compiler.run((err, stats) => {});
```
从上面的代码可以看出 webpack 是个函数并返回一个 Compiler 实例，接着运行 run 方法由此打开了潘多拉的盒子。

## webpack 函数

1. 在调用 webpack 函数时传入我们的配置文件中导出的配置项
2. 合并传入的配置项和命令行传入的配置项（写过nodejs命令行工具应该知道如何接收命令行参数）
3. 实例化 Compiler 并传入合并后的配置项
4. 注册配置项传入的所有插件 `plugin`，这里只是遍历执行 `plugin` 上的 apply 方法，plugin 的真正意义是在 webpack 编译过程中每个阶段 `hooks` 做扩展，调用 apply 方法就是注册监听 webpack 中钩子函数
5. 返回 Compiler 实例

## compiler 类
compiler 的定义是个 class 

1. 构造函数内定义了一些属性，包括传入的配置项也被赋值给了私有属性，另外还初始化了一些 hooks 实例
2. 重点是 run 方法，在执行 run 方法时就开始执行编译了
    
    * 在传入的配置项中 entry 获取到入口文件
    * 从入口文件出发开始编译打包
    * 通过路径 fs 到源码
    * 获取到原始源码后首先经过 loader 的洗礼，loader 的具体操作后面的文章会详情解释
    * 经过 loader 返回的源码在给到 webpack 的 ast 遍历找到语法树中的 require 节点，从新拼接出 require 引入模块的绝对路径添加到当前模块的 dependencies 中作为当前模块的依赖项
    * 在此引出 webpack 比较重要的概念 moduleId 和 dependencies ，首先会创建一个私有变量 module 对象，module 中会有 moduleId，dependencies， _source 等属性，当前模块的相对路径就是 moduleId ，当前模块引入的其他模块的绝对路径会 push 到 dependencies中，_source 就是当前模块抽象语法树返回的新代码
    * 编译模块是递归操作
    * 接着会遍历当前模块的 dependencies 数组，递归的操作编译模块这步，也就是在从上面的第三步开始
    * 根据入口和模块之间的依赖关系，组成一个 Chunk ，Chunk 也是一个对象，包含 name， modules 等属性，比如多入口的应用或者异步加载模块都产生不同的 Chunk
    * 最后通过依赖关系组成一个 Chunk 被 push 到 Chunks 中，最后的最后就是遍历 Chunks 通过 fs 写入到磁盘
 
## 总结
1. 初始化参数：从配置文件和 Shell 语句中读取并合并参数，得出最终的配置对象
2. 用上一步得到参数初始化 Compiler 对象
3. 加载注册所以配置的插件
4. 执行 run 方法开始编译
5. 根据配置中的 entry 找到入口文件，也可能是多入口
6. 从入口文件出发，调用所用配置的 Loader 对模块进行编译
7. 再找出该模块的依赖模块，再递归模块编译这一步骤
8. 根据入口文件和模块直接的依赖关系，组装成一个 Chunk
9. 再把每个 Chunk 转换成一个单独文件添加到输出列表
10. 根据配置确定输出的路径和文件名，把文件写入到磁盘