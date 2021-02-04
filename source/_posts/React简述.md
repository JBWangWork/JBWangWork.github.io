---
title: React简述
date: 2019-12-05 11:33:55
author: Vincent
categories: 
- 前端
tags: 
- React
---


![React简述](https://upload-images.jianshu.io/upload_images/5741330-d92ca6829b68471e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> React是Facebook在2013年推出的前段框架。React不是一个MVC框架，React是一个构造可组合式用户界面的库。它鼓励创建可重用的UI组件会随着时间而改变的数据。React采用不同的方法，当组件第一次初始化时，render方法调用，为试图生成一个轻量级的表现。通过这个表现，产生一个标签字符串，然后插入文档中。当数据变化时，render方法再次被调用。为了尽可能有效的完成更新，我们比较之前调用的render返回的值与新的值，然后产生一个最小的变更去应用DOM中。

**问题出现的根源**
- 传统 UI 操作关注太多细节
- 应用程序状态分散在各处难以追踪和维护

Facebook认为MVC无法满足他们的扩展需求，由于他们非常巨大的代码库和庞大的组织，使得MVC很快变得复杂，每当需要添加一项新功能或者特性时，系统的复杂就成级数的增长，致使代码变得脆弱而不可预测，结果导致他们的MVC正在土崩瓦解。认为MVC不适合大规模的应用。当系统中有很多模型和相应的视图时，其复杂度就会迅速扩大，非常难以理解和调试，特别是模型和视图可能存在双向数据流动。

React：始终整体"刷新"页面，无需关心细节。以组件的方式去描述UI，4个必须API，单向数据流，完善的错误提示。

#### 数据模型如何解决

传统MVC难以扩展和维护，Model和View之间的关系错综复杂而且双向绑定，如果业务复杂出现问题后很难去追踪是Model的问题还是View的问题。

![传统MVC难以扩展和维护](https://upload-images.jianshu.io/upload_images/5741330-b68448c8281d3033.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

针对这个问题，React提出Flux架构，该设计模式的核心思想就是单向数据流。首先用户操作产生Action，Action经过Dispatcher再发送到Store，Store根据Action来进行处理，View是绑定到Store上的，此时View随Store更新而更新。
![Flux架构：单向数据流](https://upload-images.jianshu.io/upload_images/5741330-84f6f39938abd0a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Flux架构的衍生项目：Redux、Mobx。

#### 组件

React组件由props和state最终得到一个View。状态有外部传过来状态和内部维护状态，两种状态最终决定了View。React一般不提供方法，而是某种状态机，可以理解React组件为一个纯函数，React是单向数据绑定。
以组件的方式考虑UI的构建，将UI组织成组件树的形式。

下面的评论框由CommentBox、CommentList和CommentForm三个组件共同完成，代码部分除了div和h1标准的html的tag外还有CommentList和CommentForm两个自定义组件共同完成。

![UI的构建](https://upload-images.jianshu.io/upload_images/5741330-d4117fdca7057b4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

受控组件的状态来自外部，要传递value和onChange，非受控组件的状态由内部维护，如果外部需要可以通过其他方式获取，也就不需要传递value和onChange。
![受控组件和非受控组件](https://upload-images.jianshu.io/upload_images/5741330-5044d8d3c4000229.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在创建组件时要遵守单一职责原则。每个组件只做一件事，组件是构建UI的最小元素，每个组件都应该尽量的小，这样才能够让复杂度分散出去，在以后的开发中如果组件变得复杂，应该拆分成小组件。
数据状态管理要遵守DRY原则。能计算得到的状态就不要单独存储；组件尽量无状态，所需数据通过props获取以提高组件的性能。

#### JSX的本质  

JSX：在JavaScript代码中直接写HTML标记。JSX并不是模板语言，只是一种语法糖。TIP：自定义组件以大写字母开头，React认为小写的tag是原生DOM节点，如div；JSX标记可以直接使用属性语法，例如`<menu.Item />`。

比如我们定义了一个变量name，又定义了一个HTML的element：
```
const name = 'Vincent Wang';
const element = <h1>Hello, {name}</h1>
```

实际上是动态创建了一个组件，而且是用JavaScript语法。
![image.png](https://upload-images.jianshu.io/upload_images/5741330-dc3e9bcfb0456cb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

JSX本身也是表达式，element返回一个HTML的节点：
```
const element = <h1>Hello, world!</h1>
```
在属性中使用表达式，如果给一个组件传递属性的值，这个属性的值可以是JavaScript的表达式：
```
<MyComment foo={1 + 2} />
```
延展属性，如果给一个组件传递一组值，此时我们不需要一个一个填写，只需要`...`语法，
```
const props = { name: "Red", value: "red" };
const greeting = <Greeting {...props} />;
```
表达式作为子元素，子元素是React的一个特殊属性。只需要用大括号将JavaScript语法包起来。
```
const element = <li>{props.message}</li>
```

** JSX的优点**
- 有声明式创建界面的直观
- 代码动态创建界面的灵活
- 不需要学习新的模板语言

#### React生命周期及使用场景

React主要分三个阶段：Render阶段（计算当前的状态）、Pre-commit阶段（读取DOM内容）和Commit阶段（React把当前状态映射到DOM节点时需要实际更新DOM节点）。

![生命周期](https://upload-images.jianshu.io/upload_images/5741330-6a348940407ff2af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图片来源：[http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)

生命周期可以分为三个类型：挂载时、更新时和卸载时。

##### 创建时
1. constructor
一个组件的构造函数的创建，也是唯一可以直接修改state的地方。
2. getDerivedStateFromProps
从外部的属性初始化内部的状态，在初始安装和后续更新上，都在调用render方法之前立即调用getDerivedStateFromProps。它返回一个对象以更新状态，或者返回null则不更新任何内容。这个方法一般不建议使用，如果state需要从props中获取的时候，可以通过props动态计算等到，不需要单独存储这个状态，否则要维护两者状态一致性，这样会增加复杂度，每次render都会调用。一般表单控件获取默认值。
3.render
描述UI DOM结构，类组件中唯一需要的方法。
4. componentDidMount
挂载组件（插入树中）后立即调用`componentDidMount（）`。我们可以在此处发起请求或者定义外部的资源等。只执行一次，可以获取外部资源。

##### 更新时
当组件有新的属性或者修改了state内部状态，当然如果强制刷新`forceUpdate`可以会触发。
1. getDerivedStateFromProps
同样是从外部属性得到内部的状态
2. shouldComponentUpdate
使用shouldComponentUpdate（）让React知道组件的输出是否不受当前状态或道具更改的影响。在这里可以做一些组件的性能优化，比如props即使发生变化且UI不需要更新，这时可以通知React组件直接返回false则不需要Update。一般可以由PureComponent自动实现，来判断当前state和props是否发生变化。
3. render
计算虚拟的DOM，虚拟DOM维持内部的UI状态，计算diff等。
4. getSnapshotBeforeUpdate
它使组件可以在DOM可能发生更改之前从DOM捕获一些信息（例如，滚动位置），返回的任何值都将作为参数传递给componentDidUpdate（），一般在获取render之前的DOM状态时使用。
5. componentDidUpdate
发生更新后，立即调用componentDidUpdate（），初始渲染不调用此方法。React更新的事情做完后可以根据实际业务在此处做一些处理，然后更新到UI上。页面根据props变化来重新获取数据。

##### 卸载时
- componentWillUnmount
当组件移除时，需要销毁组件来释放资源。

该文章为记录本人的学习路程，也希望能够帮助大家，知识共享，共同成长，共同进步！！！

