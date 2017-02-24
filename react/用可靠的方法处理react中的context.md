#如何以可靠的方式处理react中的context [原文](https://medium.com/react-ecosystem/how-to-handle-react-context-a7592dfdcbc#.24nk6l24c) 

## Context 还未

react中的单向数据流的特性，可以使得你的应用中的改变有迹可循。你可以准确知道哪个组件正在传递props，你甚至可以通过采用容器组件或者木偶式组件来展现。
当你用了一段时间的react之后，你就会遇到一个问题。那就是在组件往下传递数据时，你只能一层一层往下传播，而不能跨层传播。
react 添加了一个新特性context来解决了这个问题。
你可能已经享受到这个特性带来的好处，即使你没意识得到。在react生态系统中，有很多库都用到了它，例如：React-Router, React-Redux。
但是，使用它必须小心。react 文档警告我们 context是一个不稳定的api，可能在以后移除。
这篇文章旨在向您展示如何可靠地使用context。
接下来简要说明最常见的陷阱以及如何避免它们。

## context的工作原理
首先你需要一个提供context的组件。这只是一个包含`childContextTypes`和`getChildContext`的普通组件。
* `childContextTypes`:**`static property`** 允许你声明一个传入子组件的context对象结构。有点类似`propTypes`，但是必须声明。
* `getChildContext`:**`prototype method`** 通过返回一个context的对象来向组件传递。当state改变或者组件接受一个新的props的时候这个方法将被调用

看看下面例子
```
class ContextProvider extends React.Component {
  static childContextTypes = {
    currentUser: React.PropTypes.object
  };
  getChildContext() {
    return {currentUser: this.props.currentUser};
  }
  
  render() {
    return(...); 
  }
}
```

这个组件的任何子组件都可以接受context

```
/* ES6 class component */
class ContextConsumer extends React.Component {
  /* 
     contexTypes 需要一个 static property来声明你想要从context获得的东西
  */
  static contextTypes = {
      currentUser: React.PropTypes.object
  };
  render() {
    const {currentUser} = this.context;
    return <div>{currentUser.name}</div>;
  }
}
/* ES6 class component with an overwritten constructor */
class ContextConsumer extends React.Component {
  static contextTypes = {
      currentUser: React.PropTypes.object
  };
  constructor(props, context) {
    super(props, context);
    ...
  }
  render() {
    const {currentUser} = this.context;
    return <div>{currentUser.name}</div>;
  }
}
/* Functional stateless component */
const ContextConsumer = (props, context) => (
  <div>{context.currentUser.name}</div>
);
ContextConsumer.contextTypes = {
  currentUser: React.PropTypes.object
};
```

`context consumer components`需要通过`contextTypes`来声明，这样才能获取context。然后，在ES6类组件的情况下，他们可以通过this.context访问上下文。相反，`consumer`是一个无状态组件时，您可以通过函数的第二个参数访问context

此外，您可以在一些组件的生命周期方法中引用context
```
componentWillReceiveProps(nextProps, **nextContext**) {...}
shouldComponentUpdate(nextProps, nextState, **nextContext**) {...}
componentWillUpdate(nextProps, nextState, **nextContext**) {...}
componentDidUpdate(previousProps, **previousContext**) {...}
```

##缺点
1. 正如我们上面提到的，context是实验性的，API在未来可能会改变。这使得使用它的代码过于脆弱。任何API更改都会破坏每个context,这会使得之后维护变得非常困难。
2. 此外，context基本上是单个React子树范围内的全局变量。这使得你的组件更加耦合，并且几乎不可能在上述子树之外重用.
3. 还有，中间层的组件中 shouldComponentUpdate 返回false，如果组件提供的context更改，使用该值的中间层的子组件将不会更新，并返回false。

![](/img/context.png) 

哇！有很多要考虑...和一个问题出现
我应该使用React不稳定的context功能吗？

让我们看看如何使用高阶组件（HOCs为简称），你可以减轻一些使用React语境的麻烦

< 关于高阶组件
< [Mixins Are Dead. Long Live Composition](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750#.yg1n4cm3u)
< [Higher Order Components: A React Application Design Pattern](https://www.sitepoint.com/react-higher-order-components/)
< [Mixins Considered Harmful](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html)

提供HOCs可以轻松解决context的代码脆弱性问题。如果API更改，你只需更新一个位置。下面来实现这个高阶组件
```
// 传入  childContextTypes，getChildContext 和原来的写法一样
const provideContext = (childContextTypes, getChildContext) => (Component) => {
    class ContextProvider extends React.Component {
      static childContextTypes = childContextTypes;
      //
      getChildContext = () => getChildContext(this.props);
        
      render() {
        return <Component {...this.props} />;
      }
    }
    return ContextProvider;
  };

const consumeContext = (contextTypes) => (Component) => {
  /* context转变为props传递，这样完全和context API解耦了
  */ 
  const ContextConsumer = (props, context) =>
    <Component {...props} {...context} />;
  ContextConsumer.contextTypes = contextTypes;
  return ContextConsumer;
};
```
你可以像下面一样使用
```
const Child = ({color}) => (
  <div style={{backgroundColor: color}}>
    Hello context!!!
  </div>
);
const ChildwithContext = consumeContext({
  color: React.PropTypes.string
})(Child);
const MiddleComponent = () => <ChildwithContext />;
const App = provideContext(
  {color: React.PropTypes.string},
  () => ({color: 'red'})
)(MiddleComponent);

```
另一个选择是使用[Recompose](https://github.com/acdlite/recompose)
Recompose提供了一堆非常有用的HOC，以满足你的所有需求。 如果你接受Recompose，你将改变你编写React应用程序的方式。 它分别提供withContext和getContext作为我们的provideContext和consumeContext的替代
作为优势，您不再需要维护HOC。但是，如果你的应用程序很小，或者你只需​​要这两个功能，依靠Recompose可能是浪费了

##不要使用context向组件传递模型数据
**HOCs给你很多，但不够...它由你决定通过context传递什么数据**
如果你传递模型数据，你只会得到更多的麻烦
1. 让你的应用程序更难追踪状态变化
2. 组件无法解耦

使用context实现全局变量，而不仅仅是保存输入数据。例如，当前登录的用户信息，主题信息，国际化和本地化都是可以使用context来保存

##结论
在软件开发中，就像在生活中一般，你需要权衡各种利弊。contxt是一个强大的工具，虽然不稳定。