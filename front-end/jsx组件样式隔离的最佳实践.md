## 前言

在当今前端开发中，组件化研发模式已然是大行其道，各种基于组件化的搭建系统更是层出不穷，在提升业务研发效率的同时，组件化面临着一个痛点—组件样式隔离的问题。  

一个组件可能会被多个业务页面所使用，如果不做任何处理，业务在使用该组件的过程中就极有可能会发生组件与组件或者组件与页面因为class命名撞衫而导致的样式覆盖问题，最终使页面展示异常。  

针对这个问题目前业界也已经有了很多成熟的方案，包括 css module, css in js 以及BEM命名约定等等。但是这些方案在编码体验和最终构建产物上都或多或少的存在一些问题。  

以最为典型和常用的 css module 方案为例，它需要在 jsx 中进行 className 的动态绑定，导致的问题是需要先编写样式而不是先编写元素结构和定义 className。另外一个问题是在 className 写法上由于需要使用获取对象属性的写法，会导致一些使用连字符的样式类名需要用中括号才行，比如如下代码：
  
  ``` css
  .container-title {
    color: red;
  }
  ```

  ``` jsx
  import React from 'react';
  import style from './App.css';

  export default () => {
    return (
      <h1 className={style["container-title"]}>
        Hello World
      </h1>
    );
  };
  ```
  这种编码方式确实算不上优雅，毕竟对于前端开发者来说最爽最熟悉的肯定还是直接编写 className 字符串，然后在 css 文件中去编写对应 class 的样式。此外其编译产物中 className 的值会变成一个哈希字符串，如下所示：
  ```html
  <h1 class="_3zyde4l1yATCOkgn-DBWEL">
  Hello World
  </h1>
  ```
  ```css
  ._3zyde4l1yATCOkgn-DBWEL {
    color: red;
  }
  ```
  虽然类名确实变成独一无二了，但是可读性极差并且如果在作为其他组件的子组件使用时，如果父组件想要覆盖子组件样式，这种情况下就没法儿支持了。  

其他的像 css in js 这种需要在 js 中编写样式，这本身就不太符合关注点分离的开发习惯，不仅会导致js文件的膨胀，并且其构建产物中样式大多是通过 style 内联的形式，这种方式对于样式复写也会造成较高的成本。  

铺垫做了这么多，接下来会介绍我对于解决 jsx 组件样式隔离问题的最佳实践，它来源于我自己思考并开发的两个 webpack-loader。  

## 最佳实践

### 示例展示

你需要在组件研发脚手架的 webpack 配置中添加 [scope-jsx-loader](https://github.com/NealST/jsx-style-scoped) 和 [scope-css-loader](https://github.com/NealST/jsx-style-scoped)。使用示例如下： 

先完成 loader 安装
```shell
npm i scope-jsx-loader scope-css-loader --save-dev
```
然后可以在 webpack 中进行 loader 添加
```javascript
module.exports = {
  module: {
    rules: [
      {
        test: /\.(t|j)sx$/i,
        exclude: /node_modules/,
        use: [
          {
            loader: 'babel-loader'
          },
          {
            loader: 'ts-loader'
          },
          {
            loader: 'scope-jsx-loader'
          }
        ],
      },
      {
        test: /\.(c|sc|sa)ss$/i,
        use: [
          {
            loader: 'style-loader',
          },
          {
            loader: 'css-loader',
          },
          {
            loader: 'sass-loader',
          },
          {
            loader: 'scope-css-loader'
          }
        ],
      }
    ]
  }
}
```
[scope-jsx-loader](https://github.com/NealST/jsx-style-scoped) 负责对 jsx 文件进行解析和转换，它会查找 jsx 中所有的 className, 然后将每个 className 的值转换为 `${className}-${hash}` 的模式。  

[scope-css-loader](https://github.com/NealST/jsx-style-scoped) 负责同步样式文件中对应类名的变更，将类名选择器转换为 `.${className}-${hash}`

这个过程完全是在构建环节自动进行的，你不需要像 css module 那样关注 jsx 和样式文件关联的细节，可以正常编写 className 和样式文件，以一个 rax 组件为例：

jsx 文件代码：
```jsx
import { createElement } from 'rax';

import View from 'rax-view';
import Text from 'rax-text';
import Image from 'rax-image';

import './index.scss';

interface ComponentData {
  bgColor: string;
}

interface PropsData {
  fields: ComponentData;
}

const Demo = (props: PropsData) => {
  const { bgColor } = props.fields;

  const style = {
    backgroundColor: bgColor,
  };

  return (
    <View className="component-container" style={style}>
      <Image className="container-img" source={{ uri: 'https://gw.alicdn.com/tfs/TB1LYpTL1L2gK0jSZFmXXc7iXXa-260-260.jpg' }} />
      <Text className="container-text">Welcome to develop a component</Text>
    </View>
  );
};

export default Demo;
```
index.scss 文件代码：

```scss
.component-container {
  background-color: #fff;
  .container-img {
    width: 100rpx;
    margin: 60rpx auto;
    height: 100rpx;
  }
  .rax-view{
    font-size: 12px;
  }
  .container-text {
    width: 100%;
    text-align: center;
    font-size: 24rpx;
    font-weight: 500;
  }
}
```
其编译生成的 html 代码效果如下所示：
![](https://img.alicdn.com/tfs/TB1jTGZPET1gK0jSZFrXXcNCXXa-858-338.png)

css 代码效果如下所示：
![](https://img.alicdn.com/tfs/TB1VwF6d9R26e4jSZFEXXbwuXXa-812-556.png)

通过 `${className}-${hash}` 的方式，我们就完成了组件样式的隔离，确保了不会发生类名全局污染的问题。整个过程对于开发者是无感的，他们可以用最简洁的开发方式来编写代码。接下来我会为你解析整个实现过程的内在原理。

### 过程解析

上述的 hash 值由 md5 根据当前组件的 npm 包名进行生成，为了避免增加过多字符串导致组件的包体积大幅增加，我只取了 hash 字符串的前8位字符。这种方式既保障了类名的可读性，也保障了组件的类名唯一性。hash 生成代码如下所示：
```javascript
const md5 = require('md5');

const path = require("path");

const computedHash = {};

const computeHash = (pkgName) => {
  if (computedHash[pkgName]) {
    return computedHash[pkgName]
  }
  const hash = md5(pkgName).substr(0, 8);
  computedHash[pkgName] = hash;
  return hash;
}
const cwd = process.cwd();
const pkgName = require(path.join(cwd, 'package.json')).name;
const hash = computeHash(pkgName);
```

这里需要解释一下的是为什么使用组件的 npm 包名而不是 jsx 文件的路径来生成 hash。如果使用文件路径的方式，本地构建生成的 hash 字符串和云端构建生成的 hash 字符串会不一致，同一个组件被多个人协作开发时不同开发者在本地构建生成的 hash 字符串也会不一致，这种不一致带来的后果就是当该组件作为子组件嵌入到父组件中使用时，父组件由于无法确定子组件的类名，就无法完成对子组件的样式复写。而使用 npm 包名就不存在这个问题了，对于一个组件来说，其 npm 包名是唯一的，这样其生成的 hash 值也是唯一的，且不会发生变化。

完成 jsx 文件中 className 值的修改之后，还需要将上文中生成的 hash 值传递给样式文件，完成样式文件中相关类名的修改。这里会涉及到两个问题：
* 如何传递 hash 值
* 如何确定样式文件中哪些类名需要修改

第一个问题可以通过给 jsx 中引入的样式文件添加查询字符串的方式来解决，示例代码如下：

```javascript
const styleReg = /\.(c|sc|sa|le)ss/g;
return source.replace(styleReg, (match) => {
  return `${match}?scopeId=${hash}`;
});
```  
在 [scope-css-loader](https://github.com/NealST/jsx-style-scoped) 中会解析 scopedId 的参数来获取哈希值。  

第二个问题的解法其实也非常简单，一共分为两步。第一步，先统计jsx文件中有哪些 className, 这里需要注意的一点是 className 的编写是可以支持多个类名以空格形式组合的，比如：
```html
<h1 className="hello1 hello2" />
```
在这种写法下 h1 这个元素其实是有 hello1 和 hello2 两个类名，需要单独进行收集，且每个类名都需要单独添加 hash，考虑到添加 hash 的过程本身也需要查找类名，类名的统计和替换可以放在一起做，代码如下所示：  
```javascript
const classNameReg = /className=\"([^"]+)\"/g;
// 负责收集需要转换的样式类名
let classnames = [];
return source.replace(classNameReg, (match) => {
  const classValues = match.match(/className=\"([^"]+)\"/)[1].trim().split(" ");
  // 转化成带.的选择器
  classnames = classnames.concat(classValues.map(item => `.${item}`));
  return `className="${classValues.map(item => `${item.trim()}-${hash}`).join(" ")}"`;
})
```
这里有个问题需要说明一下，为什么我们需要做这一步的收集工作。可能有人会问，样式文件中写的类名不应该都是我在 jsx 文件中编写的 className 吗？我难道不能在样式文件中全量给所有的类名都添加 hash 值吗？答案是当然不能，因为在样式文件中是有可能存在以下代码的：
```css
div{
  font-size: 12px;
}
```
在这种情况下很明显是不能给 div 添加 hash 值的。可是既然这样那过滤一下选择器就好了啊，只给用类名的选择器添加 hash 不就可以了吗？答案也是不行，因为还有可能会存在下述这种代码：
```css
.container{
  .next-btn{
    color: #fff;
  }
}
```
.next-btn 可能是你在组件中使用的一个 fusion 或 antd 的按钮组件，你想覆盖其样式，所以写了这么一行代码。这种情况下. next-btn 很明显也是不应该添加 hash 的，这样会导致你需要的样式覆盖失效。  

基于上述原因我们不能无脑的对样式文件中的类名进行全局替换，需要进行这一步的收集工作。  

接下里是第二步，需要将收集到的类名传入到样式文件中。这个也很简单，可以借鉴 hash 值传递的方式，在之前的基础上添加一个 classnames 参数即可，代码如下所示：

```javascript
const styleReg = /\.(c|sc|sa|le)ss/g;
return source.replace(styleReg, (match) => {
    return `${match}?scopeId=${hash}&classnames=${classnames.join('($$)')}`;
});
```

这时候 jsx 中样式文件的引用就由
```jsx
import './index.scss'
```
变成了
```jsx
import './index.scss?scopedId=f1954ada&classnames=.component-container($$).container-img($$).container-text'
```  

剩下的工作就是 [scope-css-loader](https://github.com/NealST/jsx-style-scoped) 对样式文件进行解析和替换。这个过程也是分为两步。  

第一步，解析 scopeId 和 classnames 参数，代码如下所示：
```javascript
const qs = require('qs');
const resourceQuery = qs.parse(this.resource.split('?')[1]);
const scopeId = resourceQuery.scopeId;
const classnames = resourceQuery.classnames && resourceQuery.classnames.split('($$)');
```

第二步，使用 scopeId 和 classnames 进行类名替换，这里需要注意的一点是在获取样式文件中的类名时需要考虑到后代选择器的场景，比如：
```scss
.a .b{
  color: #fff;
}
```
在这种场景下，a 和 b 是两个独立的类名，需要单独进行处理。类名解析和替换的代码如下所示：
```javascript
const classNameReg = /\.([^{]+)(\s*)\{/g;
if (scopeId && classnames) {
  return source.replace(classNameReg, (matchItem) => {
    const theClassName = matchItem.match(/\.([^{]+)(\s*)\{/)[1].trim();
    // 兼容css的后代选择器模式，比如 .a .b{}
    const classValues = theClassName.split(/(\s+)/);
    const ultiClassName = classValues.map((item, index) => {
      const checkValue = index === 0 ? `.${item}` : item;
      // 判断是否在需要替换的类名名单中
      return classnames.indexOf(checkValue) >= 0 ? `${checkValue}-${scopeId}` : checkValue;
    }).join(" ");
    return `${ultiClassName} {`;
  })
}
```

至此所有的工作就完结了，整个过程其实没有特别难的地方，核心还是需要考虑和兼容开发者的各种编码场景，比如上文中提到的后代选择器以及 className 中的多类名问题等等，这里有其他考虑不周全的地方也欢迎大家进行指正。  

## 结语

本文主要介绍了我对于 jsx 组件进行样式隔离的最佳实践，目前该方案已经集成到了我们团队内的组件脚手架中。如果你有其他更好的方案也欢迎随时与我进行交流探讨，后续我也会将这部分能力支持到其他构建工具。  

我们是业务平台-体验技术团队，目前正在全力打造全新的阿里巴巴业务中台基础设施，不论是技术深度还是业务场景都有非常大的挑战，欢迎各位前端或者后端大佬的加入。  

联系方式：
微信：longmaost
邮箱：mozheng.sh@alibaba-inc.com