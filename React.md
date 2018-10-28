## React 生命周期

1. **componentWillMount** 在**渲染前调用**,在客户端也在服务端。
2. **componentDidMount** : 在**第一次渲染后调用**，只在客户端。之后组件已经生成了对应的DOM结构，可以通过this.getDOMNode()来进行访问。 如果你想和其他JavaScript框架一起使用，可以在这个方法中调用setTimeout, setInterval或者发送AJAX请求等操作(防止异步操作阻塞UI)。
3. **componentWillReceiveProps** 在组**件接收到一个新的 prop (更新后)时被调用**。这个方法在初始化render时不会被调用。
4. **shouldComponentUpdate** 返回一个布尔值。在组**件接收到新的props或者state时被调用**。在初始化时或者使用forceUpdate时不被调用。 
   可以在你确认不需要更新组件时使用。
5. **componentWillUpdate**在**组件接收到新的props或者state但还没有render时被调用**。在初始化时不会被调用。
6. **componentDidUpdate** 在组件**完成更新后立即调用**。在初始化时不会被调用。
7. **componentWillUnmount**在组件从 **DOM 中移除之前立刻被调用**。


### 1. componentWillMount  组件将要挂载

> 1、组件刚经历constructor,初始完数据
>  2、组件还未进入render，组件还未渲染完成，dom还未渲染

componentWillMount 一般用的比较少，更多的是用在服务端渲染，（我还未使用过react服务端渲染哈，所以也写不了很多）
 **但是这里有一个问题**

> ajax请求能写在willmount里吗？
>  ：答案是不推荐，别这么写

1.虽然有些情况下并不会出错，但是如果ajax请求过来的数据是空，那么会影响页面的渲染，可能看到的就是空白。
 2.不利于服务端渲染，在同构的情况下，生命周期会到componentwillmount，这样使用ajax就会出错

###2.componentDidMount 组件渲染完成

组件第一次渲染完成，此时dom节点已经生成，可以在这里调用ajax请求，返回数据setState后组件会重新渲染

 



React组件生命周期有三个阶段：加载、更新和卸载。每个阶段有多个方法来调用实现某些功能。这些方法名字也很有意思，带will前缀表示在该阶段发生之前调用，did表示在该阶段发生之后调用。本文将介绍这些方法。本文翻译自[React官网文档](https://facebook.github.io/react/docs/react-component.html#constructor)，如有翻译不当，请不吝指正。 

1. **Mounting阶段**：该阶段表示一个组件实例被创建并被插入到DOM中。该阶段有四个方法：`constructor()`，`componentWillMount()`，`render()`和`componentDidMount()`。

- **constructor(props)**：该方法在组件加载前调用。当组件作为`React.Component`的子类实现时需要在其他声明之前先调用`super(props)`。否则，`this.props`在构造器中会是undefined，这会导致代码出现bug。 
   **作用**：用来初始化状态，如果既不初始化状态也不绑定方法，那就没必要实现该方法了。（笔者注：事实上，如果组件没有构造器方法的话，组件会默认调用一个构造器，实现属性的传递）。
- **componentWillMount()**：该方法会在加载发生前调用。因为它发生在`render()`方法前，因此在该方法内同步设置状态不会引发重渲染。该方法还是服务器端渲染的唯一的生命周期钩子。一般，推荐用`constructor()`代替该方法。
- **render()**：该方法必须要有。当调用时，它会检查`this.props`和`this.state`，然后返回一个React元素。这个元素可以是原生DOM组件如div，也可以是一个自定义的复合组件。如果什么也不想渲染的话，可以返回`null`或`false`。当返回`null`或`false`时，`ReactDOM.findDOMNode(this)`会返回`null`。该方法应该是纯净的，这意味着它不能修改组件状态，每次调用会返回相同的结果，不会和浏览器发生直接的交互。如果想要同浏览器发生交互的话，应该在`componentDidMount()`中实现。
- **componentDidMount()**：组件加载完成后触发。 
   作用：放置必要的DOM节点。如果要从远程节点加载数据，这是一个不错的实例化网络请求的地方。也可以在此处设置定时器等。 
   注意：在该方法内设置状态会导致重渲染。

2. **Updating阶段**：该阶段表示由状态或属性的改变导致组件的重渲染。该阶段有五个方法：`componentWillReceiveProps()`，`shouldComponentUpdate()`，`componentWillUpdate()`，`render()`和`componentDidUpdate()`。

- **componentWillReceiveProps(nextProps)**：该方法会在加载好的组件在收到新的状态后调用。如果需要更新状态以反映属性的改变，可以在比较`this.props`和`nextProps`后，使用`this.setState()`方法来实现状态的过渡。 
   **注意**：React可能会在props值并未改变的时候调用该方法，所以要确保每次处理改变时都要比较当前props和收到的props值。这可能发生在父组件导致该组件重渲染时。 
   **非触发时期**：mounting阶段不会调用该方法。只有在部分props属性更新时调用该方法，另外调用`this.state()`不会触发该方法。
- **shouldComponentUpdate(nextProps, nextState)**：该方法用来告诉React，组件输出是否受到当前状态或属性的影响。默认情况下，每次状态改变都会导致重渲染，在大多数情况下，你只需保持该默认行为即可。该方法在收到新的props或state时会被调用，且调用是在重渲染前。该方法默认返回`true`。但返回`false`不会影响其子组件在状态改变时的重渲染。 
   **非触发时期**：初次渲染或使用`forceUpdate()`时不会调用该方法。 
   **注意**：就目前而言，如果该方法返回`false`，以后的`componentWillUpdate()`，`render()`和`componentDidUpdate()`都不会再调用了。但未来可能该方法返回的结果只是作为暗示，而非指令，返回`false`可能仍会导致重新渲染。 
   最后，如果找到了导致性能低下的组件，可以使它继承`React.PureComponent`。该组件用props和state的浅比较实现了`shouldComponentUpdate()`。也可以自己实现该方法，通过`this.props`同`nextProps`比较，`this.state`同`nextState`比较，然后返回`false`以跳过更新。
- **componentWillUpdate()**：该方法在收到新属性和状态渲染前调用。可以用它来做更新前的准备工作。 
   **注意**：该方法不可以调用this.setState()。如果需要更新状态以响应属性的改变，使用`componentWillReceiveProps(nextProps)`代替。 
   **非触发时期**：初次渲染不会调用该方法。`shouldComponentUpdate()`返回false不会调用该方法。
- **render()**：该方法是mount和update阶段都会使用到的方法，解释同mount阶段的`render()`。 
   **非触发时期**：`shouldComponentUpdate()`返回false不会调用该方法。
- **componentDidUpdate(prevProps, prevState)**：更新发生后会立即调用该方法。可用来在组件更新后操作DOM。另外，也可以通过比较当前属性和之前属性来决定是否发送网络请求（如，状态未改变不需要做网络请求）。 
   **非触发时期**：初次渲染不会调用该方法。`shouldComponentUpdate()`返回false不会调用该方法。

3. **Unmounting阶段**：该阶段表示组件将从DOM中移除。该阶段只有一个方法：`componentWillUnmount()`。

- **componentWillUnmount()**：该方法会在组件被销毁前立即调用。 
   **作用**：可以在方法内实现一些清理工作，如清除定时器，取消网络请求或者是清理其他在`componentDidMount()`方法内创建的DOM元素。

 

 

 

