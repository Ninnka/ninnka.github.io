---
title: React v16.3 新API浅谈
date: 2018-05-01 14:37:29
categories:
	- Web
	- JavaScript
tags:
	- React
	- Context API
	- 新版API
---

> 早在 React v16.3 发布前，就时不时能看见各路大佬对新API的分析，每次看完之后都受益匪浅 :) 。如今 React v16.3 也更新了一小段时间了，看着旧项目更新到 React v16.3后满满的 warning，我决定重新阅读新的API后对代码进行更新。（当然，可以使用官方的迁移工具，但是不亲自体验一下总觉得少了什么 :- ）

## React v15.x ~ React v16 -> React v16.3

> 这次的升级主要是几个生命周期的API改动以及加入新的 `context` api，生命周期的改动是为了适应 `React fiber`。

### 生命周期

> [组件生命周期相关-官方文档](https://reactjs.org/docs/react-component.html)

#### 新API：`static getDerivedStateFromProps()`

用于替代以前的 `componentWillReceiveProps()` 和 `componentWillUpdate()`

```jsx
class Comp extends React.Component {
  // ...
  
  state = {
    someText: 'old text',
    otherAttr: 'value',
  }
  
  static getDerivedStateFromProps(props, state) {
    // 返回一个对象，对象会用于更新state
    return {
      someText: 'a new text value'
    };
    // 返回null表示不需要更新
    /* 
     * return null;
     */
  }
  
  // ...
}
```
`getDerivedStateFromProps`被设计为了 `static` 方法，以后可以直接通过 `componentName.getDerivedStateFromProps` 调用。这个方法在组件初始化时和组件的props更新时都会被调用，方法接收了两个参数，分别是新的 `props` 和 旧的`state`。

<!-- more -->

#### 新API：`getSnapshotBeforeUpdate` 

`getSnapshotBeforeUpdate` 用于在组件更新前，获取旧的props和state。
比如说：dom 的 scroll position。[官方案例](https://reactjs.org/docs/react-component.html#getsnapshotbeforeupdate)

```jsx
class ScrollingList extends React.Component {
  constructor(props) {
    super(props);
    this.listRef = React.createRef();
  }

  getSnapshotBeforeUpdate(prevProps, prevState) {
    // Are we adding new items to the list?
    // Capture the scroll position so we can adjust scroll later.
    if (prevProps.list.length < this.props.list.length) {
      const list = this.listRef.current;
      return list.scrollHeight - list.scrollTop;
    }
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    // If we have a snapshot value, we've just added new items.
    // Adjust scroll so these new items don't push the old ones out of view.
    // (snapshot here is the value returned from getSnapshotBeforeUpdate)
    if (snapshot !== null) {
      const list = this.listRef.current;
      list.scrollTop = list.scrollHeight - snapshot;
    }
  }

  render() {
    return (
      <div ref={this.listRef}>{/* ...contents... */}</div>
    );
  }
}
```

在这个例子中，每次列表的item变化时，为了保持当前列表的滚动位置，更新前会把当前的scroll position保存起来，DOM更新完成后再改变列表的滚动位置。
你有可能会有疑问，这个用 `componentWillUpdate()` 也能实现啊。是的，普通情况下，`componentWillUpdate()` 是可以做同样的事情。
但是在 `React fiber` 架构中，组件的更新分为了两个步骤：
1. render/reconciliation
2. commit

而 `componentWillUpdate, render` 都是属于第一个阶段的，在异步渲染的情况下（React 优化了 `Async Rendering`，并将在 v17 中带来新的API），第一阶段和第二阶段必定会有延迟（可能大到无法忽略，也可能小到可以忽略），这种情况下如果对浏览器进行了拉伸改变了窗口大小，则拿到的 `scrollHeight` 就是不对的了，但是 `getSnapshotBeforeUpdate` 与 `componentDidUpdate` 都在第二阶段，这个阶段会一口气完成，中间不会有延迟。

#### 弃用：`componentWillReceiveProps() / UNSAFE_componentWillReceiveProps()`

原来的 `componentWillReceiveProps()` 现在仍然能够使用，但是会有warning提示，直到下一个大版本移除为止。
请使用 `static getDerivedStateFromProps()`。

#### 弃用：`componentWillMount() / UNSAFE_componentWillMount()`

原来的 `componentWillMount()` 现在仍然能够使用，但是会有warning提示，直到下一个大版本移除为止。
请使用 `static getDerivedStateFromProps()`。

组件初始化时的数据操作，比如异步获取数据，这些操作是建议放在 `componentDidMount()` 中操作，因为从v16开始，`componentWillMount` 有可能会被调用多次，这涉及到 `React fiber`，这里不展开讨论。

#### 弃用：`componentWillUpdate() / UNSAFE_componentWillUpdate()`

原来的 `componentWillUpdate()`，与 `componentWillMount()` 相同，在 `React fiber` 架构下，组件更新时有可能会被调用多次，已经是个不安全不稳定的API，下个大版本会移除。
请使用 `getSnapshotBeforeUpdate`。

### Official Context API

> 读过 React 官方文档的同学肯定知道在 context api，这个API在v16.3之前一直都是出于实验性状态（虽说redux就是基于这个API实现的），在v16.3中，它终于变了一个稳定的API，并且被官方强烈推荐中。
> 有同学为质疑，我已经有Redux，mobx等库了，为什么还需要一个新的content api呢？
> 其实他们并不是相互替代的关系，我们完全可以根据不同场景来判断使用哪一个。

新的 `Context API` 注重于解决这几个问题（以后想到了我再补充）：
1. 对比组件的 `props` 传递，`Context API` 不需要通过每一个组件一步步往下传递。
2. 对比Redux，可以根据父子组件或者嵌套组件之间的关系来进一步确认context。
3. 两个互相嵌套的组件提供的两个Context中，key相同的部分会冲突（**待求证**）

当然，`Redux` 对比 `Context API` ，`Redux` 清晰的代码结构，reducer/store/component分的很清楚，虽说很繁杂。

让我们来[官方的案例](https://reactjs.org/docs/context.html)

```jsx
// 通过API创建一个Context对象,默认值为'light'
const ThemeContext = React.createContext('light');

// 中间的组件不需要显示的传递
const Toolbar = (props) => {
  return (
    <div>
      <ThemedButton></ThemedButton>
    </div>
  );
}

const ThemedButton = (props) => {
  // 使用Consumer包裹需要使用Context的组件
  // React将会找到最近的Theme Provider并使用它的值
  // 在这个案例中，'theme'就是在创建Provider时设置的'dark'
  return (
    <ThemeContext.Consumer>
      {theme => <Button {...props} theme={theme}></Button>}
    </ThemeContext.Consumer>
  );
}

class App extends React.Component {
  render() {
    // 使用Context对象中的Provider组件包裹要使用的组件，并设置值为'dark'
    // 被包裹的组件无论层级多深，都可以读取到Context中存储的值'dark'
    return (
      <ThemeContext.Provider value="dark">
        <Toolbar></Toolbar>
      </ThemeContext.Provider>
    );
  }
}
```

从案例中，我们不难看出，使用 `Context API` 不外乎三个步骤
1. 创建一个Context对象并设置默认值
2. 使用Context对象的Provider组件包裹外层组件
3. 使用Context对象的Consumer组件内层组件，子组件必须是一个返回组件的方法，用于传递值，可以自定义prop名

以上三个步骤就能使用 `Context API` 的基本功能。

### 动态Context

关于如何实现[动态Context](https://codesandbox.io/s/x9vm1wm05w)，我写了一个比较简单案例，当然，也可以查看官方案例，原理是一样的。

创建一个外层的组件，在state中定义好需要属性，然后传入Provider的value，同时传入一些方法，比如案例中的 `changeTexts`。
```jsx
import React from "react";
import In from "./in";

const defaultContextValue = {
  someText1: "text1",
  someText2: "text2"
};

const SomeContext = React.createContext(defaultContextValue);

class Out extends React.Component {
  state = {
    ...defaultContextValue
  };

  changeTexts = newTexts => {
    this.setState((prevState, props) => {
      return {
        ...newTexts
      };
    });
  };

  render() {
    return (
      <SomeContext.Provider
        value={{
          someTexts: this.state,
          changeTexts: this.changeTexts
        }}
      >
        <SomeContext.Consumer>
          {textAbout => {
            return <In textAbout={textAbout} />;
          }}
        </SomeContext.Consumer>
      </SomeContext.Provider>
    );
  }
}

export default Out;
```

创建一个子组件，子组件有个按钮，点击后调用Provider中的 `changeTexts` 方法。
```jsx
import React from "react";

class In extends React.Component {
  static getDerivedStateFromProps(props, state) {
    console.log("props", props);
  }

  changeTexts = () => {
    this.props.textAbout.changeTexts({
      someText1: "new text1",
      someText2: "new text2"
    });
  };

  render() {
    return (
      <div>
        {this.props.textAbout
          ? this.props.textAbout.someTexts.someText1
          : "empty"}
        <button onClick={this.changeTexts}>按钮</button>
      </div>
    );
  }
}

export default In;
```

### createRef API

v16.3以前创建ref都是这样创建的
```jsx
class MyComponent extends React.Component {
  constructor(props) {
    super(props);

    this.inputRef = null;
  }

  render() {
    return <input type="text" ref={(node) => {
      this.inputRef = node;
    }} />;
  }

  componentDidMount() {
    this.inputRef.current.focus();
  }
}
```

v16.3后，多了 `createRef` API
```jsx
class MyComponent extends React.Component {
  constructor(props) {
    super(props);

    this.inputRef = React.createRef();
    // 使用新API创建一个Ref对象
  }

  render() {
    // ref不需要传入callback，只需要传入ref对象即可
    return <input type="text" ref={this.inputRef} />;
  }

  componentDidMount() {
    this.inputRef.current.focus();
  }
}
```

### forwardRef API

createRef API 提供了一种创建ref的新方法，但是在某些场景中，我们需要直接ref一个组件内的真实DOM而不是组件本身，过去的做法是：

```jsx
const NestComp = (props) => {
  <div>
  	<input type="text" ref={props.inputRef} />
  </div>
}

class MyComponent extends React.Component {
  constructor(props) {
    super(props);

    this.inputRef = null;
  }

  render() {
    return <NestComp inputRef={(node) => {
      this.inputRef = node;
    }}/>;
  }

  componentDidMount() {
    this.inputRef.current.focus();
  }
}
```

在过去的做法中，我们需要传递一个callback到子组件的内部，再绑定到子组件真实DOM中。

但是v16.3提供的 `forwardRef API` 提供了更简便更函数式的方法。
```javascript
const NestComp = React.forwardRef((props, ref) => {
  <div>
  	<input type="text" ref={ref} />
  </div>
});

class MyComponent extends React.Component {
  constructor(props) {
    super(props);

    this.inputRef = React.createRef();
  }

  render() {
    return <NestComp ref={ inputRef }/>;
  }

  componentDidMount() {
    this.inputRef.current.focus();
  }
}
```

### StrictMode Component （严格模式）

> [官方文档](https://reactjs.org/docs/strict-mode.html)

- 使用方法

```jsx
import React from 'react';

function ExampleApplication() {
  return (
    <div>
      <Header />
      <React.StrictMode>
        <div>
          <ComponentOne />
          <ComponentTwo />
        </div>
      </React.StrictMode>
      <Footer />
    </div>
  );
}
```

上面的例子中，`ComponentOne` 和 `ComponentTwo` 将会被检测
在严格模式下，以下几点是会被检测出来的：
1. 使用了前面提到过的旧的生命周期
2. 使用传统的字符串（callback）创建ref的方法
3. 检测意料之外的副作用
4. 使用了旧的Context API

**使用严格模式更利于我们在开发期间发现错误。**
