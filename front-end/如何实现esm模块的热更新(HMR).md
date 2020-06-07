## 前言

最近在前端工程领域出现了一些新的工程化工具，诸如尤雨溪的vite以及已在GitHub社区斩获8317个star的[snowpack](https://www.snowpack.dev/#what-is-snowpack%3F)，这些工具的优势除了内置支持vue, react等框架的运行和构建，很重要的一点是开发环境下应用的快速启动能力，snowpack的启动耗时更是号称在50ms以内。笔者找了一个简单的react项目尝试了一下，证实其所言非虚。而之所以能达到这种效果，其原理在于它直接使用了esm模块来启动应用，相比webpack来说减少了模块打包构建生成bundle的耗时。  
目前主流浏览器均已支持在script中直接使用esm模块，但是本地开发中还需要解决esm模块热更新的问题，在模块代码变更时可以快速看到页面效果。接下来我会结合snowpack中的实现源码来讲解如何完成esm模块的热更新，首先来看一个用react编写的demo。

## demo展示

项目的主要目录结构和代码如下所示：
```
|--public
   |__index.html
|--src
   |__App.css
   |__App.jsx
   |__index.css
   |__index.jsx
   |__logo.svg
```  
App.css
![](https://img.alicdn.com/tfs/TB14lolIrj1gK0jSZFOXXc7GpXa-1126-1740.png)  

App.jsx
![](https://img.alicdn.com/tfs/TB13vgoIAT2gK0jSZFkXXcIQFXa-1294-1344.png)  

index.css
![](https://img.alicdn.com/tfs/TB14xSmkcKfxu4jSZPfXXb3dXXa-1666-840.png)

index.jsx
![](https://img.alicdn.com/tfs/TB13e7KX639YK4jSZPcXXXrUFXa-874-912.png)

通过snowpack启动后，本地构建信息如下所示：
![](https://img.alicdn.com/tfs/TB1RGkkIxD1gK0jSZFsXXbldVXa-553-166.png)   

页面效果如下图所示：
![](https://img.alicdn.com/tfs/TB1cKhRJkY2gK0jSZFgXXc5OFXa-780-1468.png)

## 原理分析

### 前端逻辑
在Chrome中打开该页面的调试面板，可以看到项目的页面结构以及静态资源，如下图所示：
![](https://img.alicdn.com/tfs/TB1sqkiIqL7gK0jSZFBXXXZZpXa-1024-932.png)   
![](https://img.alicdn.com/tfs/TB1aQ7pIuL2gK0jSZFmXXc7iXXa-1228-940.png)  

从上述示例图中可以看出snowpack直接使用了入口文件index.jsx的esm模块来启动应用。在HTML中要使用esm模块，只需要在scrip标签后添加type="module"即可，关于浏览器中esm模块的使用可以参考v8引擎下的这篇文章：https://v8.dev/features/modules#other-features 这里不做详细介绍。  

除了__dist__/index.js的入口文件，HTML中还添加了/liveload/hmr.js的文件，而且index.js的入口文件中也注入了一些import.meta.hot的声明代码，此外，css类型的文件也都变成了css.proxy.js的文件。接下来我们可以来看下App.js以及index.css.proxy.js的代码，看看是否也有同样的注入以及css.proxy.js都做了啥

App.js
![](https://img.alicdn.com/tfs/TB13dlYJfb2gK0jSZK9XXaEgFXa-2048-2280.png)

index.css.proxy.js
![](https://img.alicdn.com/tfs/TB11PacJkL0gK0jSZFtXXXQCXXa-2048-1164.png)  

从上述代码中可以看出在页面初始化时，snowpack在模块原有代码基础之上还注入了一些用于模块热更新相关的代码，如下所示：
![](https://img.alicdn.com/tfs/TB1DAesJkT2gK0jSZFkXXcIQFXa-1530-588.png)
对于css模块来说，snowpack将其转换为js进行管理，css内容通过js添加style标签来处理。值得注意的是上述示例代码中的import.meta.url是esm模块内的一个全局变量，其取值为当前模块的资源路径。模块中被注入的热更新代码会使用资源路径URL将当前模块注册至客户端热更新管理中心，也就是页面html中添加的liveload/hmr.js，该模块代码如下所示：
![](https://img.alicdn.com/tfs/TB1v17pkIKfxu4jSZPfXXb3dXXa-1548-5376.png)  

hmr.js代码本身不算复杂，它的主要职责是管理页面上的esm模块，监听来自本地服务器的消息，然后根据消息类型来选择是刷新页面还是动态更新模块代码，可以用一张图来诠释hmr.js与其他模块之间的关系与运行时模块更新的逻辑：
![](https://img.alicdn.com/tfs/TB1M.OFJeH2gK0jSZFEXXcqMpXa-1079-747.png)  

每个模块通过调用hmr.js提供的createHotContext方法来注册模块，当本地服务器监听到本地代码的变更时会通过websocket向hmr.js发送消息告知哪个模块发生了改变，hmr.js获取到需要更新的模块后，通过动态import的方式向本地服务器发送获取模块最新代码的请求，本地服务器收到请求后向前端推送代码，即可完成整体的热更新链路。

对于本地启动的服务器来说，它核心要做的包括三件事情：
* 页面启动时其各个模块资源的热更新代码注入
* 监听本地代码变更，然后发送消息给hmr.js
* 响应客户端模块更新的请求，发送本地最新代码文件  

接下来通过这三点来拆解本地服务端的实现逻辑

### 本地服务端逻辑

#### 热更新代码注入的实现
本地服务器在发送代码文件至前端前通过wrapResponse方法对文件内容进行代码注入，如下所示：
![](https://img.alicdn.com/tfs/TB1nAl0eOcKOu4jSZKbXXc19XXa-1868-984.png)  

该方法内部由不同类型处理方法构成，wrapHtmlResponse负责将hmr.js添加至HTML中。
![](https://img.alicdn.com/tfs/TB1xokQJeH2gK0jSZJnXXaT1FXa-1498-588.png)  

wrapEsmProxyResponse负责处理类似之前css.proxy.js之类通过js来代理管理的模块，如下所示：
![](https://img.alicdn.com/tfs/TB1Vz.VJbj1gK0jSZFuXXcrHpXa-1682-1668.png)

wrapCssModuleResponse主要处理.module.css类型的模块，其本质逻辑跟上述wrapEsmProxyResponse方法差异不大，也是转换成js来管理，这里就不贴代码了。来看最后一个
wrapJSModuleResponse方法：
![](https://img.alicdn.com/tfs/TB1Aj3WJbj1gK0jSZFuXXcrHpXa-1632-804.png)  

以上方法就是本地服务端注入热更新代码的主要实现，逻辑都不复杂，简直可以说一目了然。  

#### 本地代码变更的监听和消息推送的实现

本地代码文件的监听snowpack采用了 [chokidar](https://www.npmjs.com/package/chokidar) 这个三方库来实现，这个库解决了node.js原生提供的fs.watch以及fs.watchFile等方法存在的一些弊端，比如：
* 不能递归监视文件树的问题
* 高cpu占用的问题
* 文件更新的事件经常会重复触发  
* 在macos上使用一些编辑器比如sublime修改代码时不会触发文件更新  

这里先不做深入探讨，有兴趣的读者可以自行搜索相关资料。在webpack-dev-server中用的也是该模块。实现文件监听的代码如下所示：
![](https://img.alicdn.com/tfs/TB1UMcRJbr1gK0jSZR0XXbP8XXa-1650-696.png)  

当文件更新触发change事件后，执行onWatchEvent方法将文件变更的消息推送到前端，代码逻辑如下图所示：
![](https://img.alicdn.com/tfs/TB1VeMZJoY1gK0jSZFCXXcwqXXa-1346-912.png)  
![](https://img.alicdn.com/tfs/TB1S5LjXc4z2K4jSZKPXXXAYpXa-1430-1056.png)

updateOrBubble方法调用hmrEngine的broadcastMessage方法来播报更新事件，同时也会遍历该模块的dependents递归调用updateOrBubble方法进行dependent更新，hmrEngine的代码如下所示：
![](https://img.alicdn.com/tfs/TB1RBA5Jbj1gK0jSZFOXXc7GpXa-1446-3792.png)  

在本地服务端，每个文件都通过hmrEngine的entry进行管理。broadCastMessage方法通过websocket发送update到前端。在接受到前端的模块更新请求
```
import(id + `?mtime=${updateID}`)
```
时，本地服务端需要响应该请求，发送模块最新代码。

#### 客户端模块更新请求响应的实现
本地服务器是通过http.createServer来启动的，模块请求响应以及文件发送的实现逻辑如下所示：
![](https://img.alicdn.com/tfs/TB1A074JXT7gK0jSZFpXXaTkpXa-2048-1308.png)  
![](https://img.alicdn.com/tfs/TB1X7U6JkL0gK0jSZFxXXXWHVXa-1582-660.png)  

到这里涉及模块热更新链路的逻辑就算完整了，本地服务端的实现其实也并不复杂，相信了解了原理之后你也可以实现一个热更新工具。

## 结语

本篇文章通过snowpack的源码介绍了esm模块热更新整体链路的实现原理。虽然直接使用esm模块可以加速应用启动，但是这也是在模块数量不多的情况下，如果模块数量超过300个，构建bundle的加载体验会比esm模块更好。  

我是阿里业务平台事业部-体验技术团队的前端，
目前我们正在招聘优秀的前端同学来一起共建阿里业务中台的前端解决方案，期待你的加入，有兴趣的同学可以通过以下方式联系到我：  
邮件：mozheng.sh@alibaba-inc.com  
微信：longmaost
