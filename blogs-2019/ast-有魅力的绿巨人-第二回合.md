## AST 抽象语法树无处不在-第二回合
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=34383004&auto=1&height=66"></iframe>
## 一，Export Default 到 Component 构造器的转换
接着上回（AST 抽象语法树无处不在-第一回合）我们继续 VUE 组件转换为小程序组件中的 JavaScript 部分的转换。
首先我们看一下要转换前后的语法树与代码如下（明确转换目标）：
###### 转换之前的 AST 树与代码
<img src="http://img11.360buyimg.com/uba/jfs/t29581/123/1031219550/7088/28f16ea1/5c05dddfN076efa20.png"/>

```js
export default {// VUE 组件的惯用写法用于导出对象模块
    data(){
    },
    methods:{
    },
    props:{
    }
}
```

###### 转换之后的 AST 树与代码
<img src="http://img11.360buyimg.com/uba/jfs/t26155/227/2594893184/9762/8d47d607/5c05dddfNe823c12a.png"/>

```js
components({//小程序组件的构造器
    data(){
    },
    methods:{
    },
    props:{
    }
})
```

通过以上转换之前和转换之后代码和 AST 的对比我们明确了转换目标就是 ExportDefault 到 Component构造器的转换，下面看一下我们是如何处理的：
###### 我们做了什么（在转换中进入到 ExportDefault 中做对应的处理）:
```js
//ExportDefault 到 Component构造器的转换
ExportDefaultDeclaration(path) {
//创建  CallExpression  Component({})
function insertBeforeFn(path) {
  const objectExpression = t.objectExpression(propertiesAST);
  test = t.expressionStatement(
      t.callExpression(//创建名为 Compontents 的调用表达式，参数为 objectExpression
          t.identifier("Compontents"),[
            objectExpression
          ]
      )
  );
  //最终得到的语法树
  console.log("test",test)
}
if (path.node.type === "ExportDefaultDeclaration") {
  if (path.node.declaration.properties) {
    //提取属性并存储
    propertiesAST = path.node.declaration.properties;
    //创建 AST 包裹对象
    insertBeforeFn(path);            
  }
  //得到我们最终的转换结果
  console.log(generate(test, {}, code).code);
```
对于 ExportDefault => Component 构造器转换还有一种转换思路 下面我们看一下：<br>
[1] 第一种思路是先提取 ExportDefault 内部所有节点的 AST ，并做处理，然后创建Component构造器，插入提取处理后的 AST，得到最终的 AST

```js
//propertiesAST 这个就是我们拿到的 AST，然后在对应的分支内做对应的处理 以下分别为 data，methods，props，其他的钩子同样处理即可
propertiesAST.map((item, index) => {
  if (item.type === "ObjectProperty") {
    //props 替换为 properties
    if (item.key.name === "props") {
      item.key.name = "properties";
    }
  } else if (item.type === "ObjectMethod") {
    //...
  } else {
    //...
    console.log("other node type", item.type);
  }
});
```
[2] 第二种思路呢，就是我们上面展示的这种，不过有一个关键的地方要注意一下：

```js
  //我把 ExportDefaultDeclaration 的处理放到最后来执行，拿到 AST 首先进行转换。然后在创建得到新的小程序组件JS部分的 AST 即可
  traverse(ast, {
        enter(path) {},
        ObjectProperty(path) {},
        ObjectMethod(path) {},
        //......其他
        ExportDefaultDeclaration(path) {
        //参见上方
        }
  })       
```

如果你想在 AST 开始处与结尾处插入，请参考：

```js
path.insertBefore(
  t.expressionStatement(t.stringLiteral("start.."))
);
path.insertAfter(
  t.expressionStatement(t.stringLiteral("end.."))
);
```

## 二，VUE 组件转换为小程序组件中的 Data 部分的处理：
关于 Data 部分的处理实际上就是：函数表达式转换为对象表达式 （FunctionExpression 转换为 ObjectExpression）
###### 转换之前的 JavaScript 代码
```js
export default {
    data(){//函数表达式
      return {
        num: 10000,
        arr: [1, 2, 3],
        obj: {
          d1: "val1",
          d2: "val2"
        }
      }
    }
}
```

###### 处理后我们得到的

```js 
export default {
  data: {//对象表达式
    num: 10000,
    arr: [1, 2, 3],
    obj: {
      d1: "val1",
      d2: "val2"
    }
  }
};
```

通过如上的代码对比，我们看到了我们的转换前后代码的变化：
###### 转换前后 AST 树对比图明确转换目标：

<img src="http://img11.360buyimg.com/uba/jfs/t28720/269/1084213324/12861/fb20d3a5/5c05e360N576b3290.png"/><br/>
<img src="http://img30.360buyimg.com/uba/jfs/t29215/312/1056174776/12311/67b44b9f/5c05e384Nb98eff0b.png"/><br/>

###### 我们对 JavaScript 动了什么手脚(亦可封装成babel插件):
```js
const ast = babylon.parse(code, {
  sourceType: "module",
  plugins: ["flow"]
});

```
```js
//AST 转换node、nodetype很关键
traverse(ast, {
  enter(path) {
    //打印出node.type
    console.log("enter: " + path.node.type);
  }
})
```

###### 我们的转换部分都尽量在一个 Traverse 中处理，减少 AST 树遍历的性能消耗

```js
//Data 函数表达式 转换为 Object
ObjectMethod(path) {
  // console.log("path.node ",path.node )// data, add, textFn
  if (path.node.key.name === "data") {
    // 获取第一级的 BlockStatement，也就是 Data 函数体
    path.traverse({
      BlockStatement(p) {
        //从 Data 中提取数据属性
        datas = p.node.body[0].argument.properties;
      }
    });
    //创建对象表达式
    const objectExpression = t.objectExpression(datas);
    //创建 Data 对象并赋值
    const dataProperty = t.objectProperty(
      t.identifier("data"),
      objectExpression
    );
    //插入到原 Data 函数下方
    path.insertAfter(dataProperty);
    //删除原 Data 函数节点
    path.remove();
  }
}    
```


## 三，CSS 部分的处理：
那 CSS 我们也是必须要处理的一部分，let try
###### 以下是我们要处理的css样本
```css
const code = `
  .text-ok{
    position: absolute;
    right: 150px;
    color: #e4393c;
   }
  .nut-popup-close{
    position: absolute;
    top: 50px;
    right: 120px;
    width: 50%;
    height: 200px;
    display: inline-block;
    font-size: 26px;
  }`;
```
###### 处理后我们得到的
```css
.text-ok {
  position: absolute;
  right: 351rpx;
  color: #e4393c;
}

.nut-popup-close {
  position: absolute;
  top: 117rpx;
  right: 280.79rpx;
  width: 50%;
  height: 468rpx;
  display: inline-block;
  font-size: 60.84rpx;
}
```
通过前后代码的对比，我们看到了单位尺寸的转换（比如：top: 50px; 转换为 top: 117rpx;）。
单位的转换( px 转为了 rpx )
###### CSS 又做了哪些处理呢？
同样也有不少的 CSS Code Parsers 供我们选择 Cssom ，CssTree等等，
我们拿 Cssom 来实现上方css代码的一个简单的转换。

```js
 var ast = csstree.parse(code);
    csstree.walk(ast, function(node) {
      if(typeof node.value == "string" && isNaN(node.value) != true){
        let newVal = Math.floor((node.value*2.34) * 100) / 100;//转换比例这个根据情况设置即可
          if(node.type === "Dimension"){//得到要转换的数字尺寸
            node.value = newVal;
          }
      }
      if(node.unit === "px"){//单位的处理
        node.unit = "rpx"
      }
  });
  console.log(csstree.generate(ast));
```
当然这只是一个 demo，实际项目中使用还的根据项目的实际情况出发，SCSS，LESS等等的转换与考虑不同的处理场景哦！
## 总结

希望，本文对大家有所帮助，在技术探索的路上，我们一往无前， Paladin 精神永存！
感谢大家的耐心阅读，也欢迎大家关注 全栈探索公众号，每周都会有技术好文推出！
发现个好东东，如下：
######  将 JavaScript 代码转化生成 SVG 流程图 js2flowchart( 4.5 k stars 在 GitHub )
当你拥有 AST 时，可以做任何你想要做的事。把AST转回成字符串代码并不是必要的，你可以通过它画一个流程图，或者其它你想要的东西。
js2flowchart使用场景是什么呢？通过流程图，你可以解释你的代码，或者给你代码写文档；通过可视化的解释学习其他人的代码；通过简单的js语法，为每个处理过程简单的描述创建流程图。
马上用最简单的方式尝试一下吧，去线上编辑看看  [js-code-to-svg-flowchart](https://github.com/Bogdan-Lyashenko/js-code-to-svg-flowchart/) [9]。
此处有必要附上截图一张。

<img src="http://img10.360buyimg.com/uba/jfs/t28051/66/739453023/25363/43b62be0/5bfd0d0aNb5f485c5.png"/>


## 扩展阅读
[1] https://astexplorer.net/

[2] https://babeljs.io/docs/en/next/babel-types.html#objectexpression 

[3] https://github.com/babel/babel/tree/6.x/packages/babel-types

[4] http://esprima.org/demo/parse.html#

[5] https://segmentfault.com/a/1190000014178462?utm_source=tag-newest

[6] https://zh.wikipedia.org/wiki/%E6%8A%BD%E8%B1%A1%E8%AA%9E%E6%B3%95%E6%A8%B9

[7] https://itnext.io/ast-for-javascript-developers-3e79aeb08343

[8] https://babeljs.io/docs/en/next/babel-types.html#callexpression

[9] https://github.com/Bogdan-Lyashenko/js-code-to-svg-flowchart









