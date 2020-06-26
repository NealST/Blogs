## 前言

我们在使用webpack的时候经常会用到[webpackbar](https://github.com/nuxt/webpackbar)或者[progress-bar-webpack-plugin](https://www.npmjs.com/package/progress-bar-webpack-plugin)之类的webpack插件通过进度条等方式来展示webpack的构建进度，以提升构建过程中的反馈体验。对于不同的插件来说，它们只是进度条的UI展示形式不同而已，但最核心的webpack构建的实时进度的数据来源却是一致的，均由webpack内部的`ProgressPlugin`这个插件提供。下面我会结合源码来讲解该插件是如何计算webpack的构建进度并将进度数据暴露给第三方的进度条插件。在阅读下文之前可以试着问下自己：如果是你，你会如何计算webpack的构建进度。

## 构建进度的计算
该插件主要根据webpack的构建阶段来定义当前进度值。webpack的构建过程分为很多不同的阶段，在每个阶段webpack都暴露了对应的事件钩子。`ProgressPlugin`正是通过这些事件钩子对每个阶段都定义了一个基础进度值，代码如下所示：  
![](https://img.alicdn.com/tfs/TB1d.PvXcVl614jSZKPXXaGjpXa-1582-1200.png)  

上述代码中的`interceptHook`方法可以先忽略，这个后续会提到。  

通过上述代码你会发现`ProgressPlugin`给 `compiler`中的每个钩子都设置了一个指定的进度值。但这些进度值还不够细致到反映webpack的详细构建过程，中间还差了0.06到0.69以及0.69到0.95两个阶段的数值。webpack构建的具体执行过程主要在`compilation`中，这两个阶段的数值由`compilation`的钩子填充。  

__0.06~0.69__
![](https://img.alicdn.com/tfs/TB12tHvaLzO3e4jSZFxXXaP_FXa-2020-1848.png)  

update方法的调用由compilation的钩子触发，如下所示：  
![](https://img.alicdn.com/tfs/TB1VBAUK7Y2gK0jSZFgXXc5OFXa-2048-1524.png)

这个阶段的主要工作是`module`,`entry`以及`dependencies`的处理和构建。换个角度从`ProgressPlugin`给该阶段设置的进度值来看，这部分工作也是webpack最耗时的地方。  

__0.69~0.95__
![](https://img.alicdn.com/tfs/TB1gITQXcVl614jSZKPXXaGjpXa-1430-2928.png)  

从上述代码中可以看出这个间隔段就完全是根据compilation的hooks来计算和指定当前的构建进度值，从hook的描述中可以看出这个阶段主要是`module`, `chunk`以及`assets`等资源的优化工作。  

基本上整个webpack构建过程的进度值就是根据上述中的 `compiler` 和 `compilation` 的hooks来设置的。  


## 进度数据的透出

webpack的构建进度确定之后剩下的任务就是将进度数据透出给第三方的进度条插件进行展示。要完成该任务需要`ProgressPlugin`完成两件事情，一是提供回调函数的切入口；二是需要能在对应的hook节点执行该回调函数进行进度的百分比值的传入。以下是这两点的实现原理

### 回调函数
`ProgressPlugin`定义了handler函数来作为回调函数切入，代码如下所示：  
![](https://img.alicdn.com/tfs/TB19TU3KYj1gK0jSZFOXXc7GpXa-1330-1812.png)

### hook劫持
hook劫持的实现非常简单，主要利用webpack hook原生提供的`intercept`方法，前文中提到的`interceptHook`方法只是对于intercept方法的封装，示例代码如下：  

![](https://img.alicdn.com/tfs/TB1aTE_K7L0gK0jSZFAXXcA9pXa-1312-1272.png)

## 结语

webpack构建的进度条实现原理就是如此简单，给每个构建阶段对应的hook设置一个进度值，然后通过handler回调和hook劫持切入到构建环节将进度信息传入回调函数，最终第三插件通过handler获取到进度值后将其展示出来。 

webpack的很多其他功能其实也没有想象中的那么复杂和高大上，通过阅读其源代码了解其实现原理后，你很可能会一拍大腿，大叹一声原来如此，这本身就是一件挺有意思的事儿。  

继续打招聘广告，阿里巴巴业务平台体验技术团队招聘资深前端以及前端技术专家，欢迎投递简历。

邮箱：mozheng.sh@alibaba-inc.com  
微信：longmaost