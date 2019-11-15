# dy
## AMD （RequireJS）
AMD全称是Asynchronous Module Definition，即异步模块加载机制。后来由该草案的作者以RequireJS实现了AMD规范，所以一般说AMD也是指RequireJS。
### RequireJS的基本用法
通过define来定义一个模块，使用require可以导入定义的模块。
```js
//a.js
//define可以传入三个参数，分别是字符串-模块名、数组-依赖模块、函数-回调函数
define(function(){
    return 1;
})
```
```js
// b.js
//数组中声明需要加载的模块，可以是模块名、js文件路径
require(['a'], function(a){
    console.log(a);// 1
});
```
RequireJS的特点
对于依赖的模块，AMD推崇依赖前置，提前执行。也就是说，在define方法里传入的依赖模块(数组)，会在一开始就下载并执行。
## CMD ( SeaJS )
CMD是SeaJS在推广过程中生产的对模块定义的规范，在Web浏览器端的模块加载器中，SeaJS与RequireJS并称，SeaJS作者为阿里的玉伯。
### SeaJS的基本用法
```js
//a.js
/*
* define 接受 factory 参数，factory 可以是一个函数，也可以是一个对象或字符串，
* factory 为对象、字符串时，表示模块的接口就是该对象、字符串。
* define 也可以接受两个以上参数。字符串 id 表示模块标识，数组 deps 是模块依赖.
*/
define(function(require, exports, module) {
  var $ = require('jquery');

  exports.setColor = function() {
    $('body').css('color','#333');
  };
});
```
```js
//b.js
//数组中声明需要加载的模块，可以是模块名、js文件路径
seajs.use(['a'], function(a) {
  $('#el').click(a.setColor);
});
```
### SeaJS的特点
对于依赖的模块，CMD推崇依赖就近，延迟执行。也就是说，只有到require时依赖模块才执行。
## CommonJS
CommonJS规范为CommonJS小组所提出，目的是弥补JavaScript在服务器端缺少模块化机制，NodeJS、webpack都是基于该规范来实现的。
### CommonJS的基本用法
```js
//a.js
module.exports = function () {
  console.log("hello world")
}
```
```js
//b.js
var a = require('./a');

a();//"hello world"

//或者

//a2.js
exports.num = 1;
exports.obj = {xx: 2};

//b2.js
var a2 = require('./a2');

console.log(a2);//{ num: 1, obj: { xx: 2 } }
```
### CommonJS的特点

- 所有代码都运行在模块作用域，不会污染全局作用域；
- 模块是同步加载的，即只有加载完成，才能执行后面的操作；
- 模块在首次执行后就会缓存，再次加载只返回缓存结果，如果想要再次执行，可清除缓存
- require返回的值是被输出的值的拷贝，模块内部的变化也不会影响这个值。

## ES6 Module
ES6 Module是ES6中规定的模块体系，相比上面提到的规范， ES6 Module有更多的优势，有望成为浏览器和服务器通用的模块解决方案。
### ES6 Module的基本用法
```js
//a.js
var name = 'lin';
var age = 13;
var job = 'ninja';

export { name, age, job};
```
```js
//b.js
import { name, age, job} from './a.js';

console.log(name, age, job);// lin 13 ninja
```
```js
//或者

//a2.js
export default function () {
  console.log('default ');
}

//b2.js
import customName from './a2.js';
customName(); // 'default'
```
### ES6 Module的特点(对比CommonJS)

- CommonJS模块是运行时加载，ES6 Module是编译时输出接口；
- CommonJS加载的是整个模块，将所有的接口全部加载进来，ES6 Module可以单独加载其中的某个接口；
- CommonJS输出是值的拷贝，ES6 Module输出的是值的引用，被输出模块的内部的改变会影响引用的改变；
- CommonJS this指向当前模块，ES6 Module this指向undefined;

> 目前浏览器对ES6 Module兼容还不太好，我们平时在webpack中使用的export/import，会经过babel转换为CommonJS规范。

## 总结：
- AMD 依赖前置，提前执行 (require.js)，语法是define，require
- CMD 依赖就近，延迟执行 （sea.js），语法是 define，seajs.use([],cb)
- CommonJs 语法 module.exports=fn或者exports.a=1; 通过require('./a1')来引入
- CommonJs 模块首次执行会被缓存，再次加载只返回缓存结果，require返回的值是输出值的拷贝（对于引用类型是浅拷贝）
- es6 module 语法是 export {...}, import ...from..., export输出的是值得引用。
- NodeJS、webpack都是基于commonJs该规范来实现的

#### 一些小问题
- 在vue项目中可能存在，我们A模块导入了B模块，C模块也导入了B模块，那么B模块在一个项目中被导入了多次的场景，模块化机制是如何处理的呢？
> 如果你是通过commonJS来require导入的模块，那么，首次导入会被缓存，后续的导入都是取的第一次的缓存结果。如果你是通过es6 import导入的模块，那么导入的都是值的引用。所以，导入多次是在最终打包的时候，只打了一个而并不会重复打入的情况。。
- 如果深入理解，先需要多使用，然后还是需要把CommonJS的内部实现，与 es6的内部实现过一遍。

##  保留个问题： B导入模块A，我们自己如何实现？不用模块化
1. 把模块A的代码复制到了模块B，显然不靠谱，这样会被复制出来的代码量太大
2. 动态创建script标签，先让A引入并执行。然后在引入B
3. es6 Module是导出的值的引用。这个是如何实现的？
4. 多次重复导入如何处理？
5. 循环引用如何处理？
6. ......
## 总结-Commonjs 和 es6 module 的区别
- Commonjs是同步导入，因用于服务端，文件都在本地，同步导入即使卡住主线程影响也不大，后者是异步导入，因为用于浏览器端，需下载文件，如果采用同步导入对渲染会有很大影响。
- CommonJS 在导出时都是值的拷贝，就算导出的值变了，导入的值也不会变。如果想更新，必须重新导入一回。
- ES Module 导入导出的值指向同一个内存地址。所以，导入值也会随着导出值变化。
- ES Module 会编译成 require/exports来执行。
- 一个导出的是值的拷贝，一个导出的是值的引用
- 一个是同步导入，一个是异步导入

参考：https://juejin.im/post/5db95e3a6fb9a020704bcd8d

码字不易，请多多指点其中的问题点，使用的文章很多，难的是，真正的理解其内部机制与如何实现的？从模块作者的角度考虑问题，多思考，学习才能掌握要领。从而实现自己的工具。

