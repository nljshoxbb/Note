#如何以可靠的方式处理react中的context [原文](https://medium.com/react-ecosystem/how-to-handle-react-context-a7592dfdcbc#.24nk6l24c  "Title") 

## Context 还未

react中的单向数据流的特性，可以使得你的应用中的改变有迹可循。你可以准确知道哪个组件正在传递props，你甚至可以通过采用容器组件或者木偶式组件来展现。
当你用了一段时间的react之后，你就会遇到一个问题。那就是在组件往下传递数据时，你只能一层一层往下传播，而不能跨层传播。
react 添加了一个新特性context来解决了这个问题。
你可能已经享受到这个特性带来的好处，即使你没意识得到。在react生态系统中，有很多库都用到了它，例如：React-Router, React-Redux。
但是，使用它必须小心。react 文档警告我们 context是一个不稳定的api，可能在以后移除。
