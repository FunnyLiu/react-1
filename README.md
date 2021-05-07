# 源码分析


## 知识点

### React.memo做了什么？怎么做的？

React.memo(...) 对应的是函数组件，React.PureComponent 对应的是类组件。

React.memo 会返回了一个纯组件 MemodFuncComponent。 我们将在 JSX 标记中渲染此组件。 每当组件中的 props 和 state 发生变化时，React 将检查 上一个 state 和 props 以及下一个 props 和 state 是否相等，如果不相等则函数组件将重新渲染，如果它们相等则函数组件将不会重新渲染。


从实现的角度来看，React.memo方法只是在组件上增加了一个标识位：[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react/src/ReactMemo.js#L27)

``` js
  //相当于带上了一个标识的type
  const elementType = {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
```

然后在Fiber阶段会针对这个类似额外做适配[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3603)：

``` js
    case MemoComponent: {

      //Memo组件会去走这个逻辑
      return updateMemoComponent(
        current,
        workInProgress,
        type,
        resolvedProps,
        updateLanes,
        renderLanes,
      );
    }
```

该方法会进行compare比较[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L490)：

``` js
    // 如果两次props相等
    if (compare(prevProps, nextProps) && current.ref === workInProgress.ref) {
      //直接返回，不再进行下面的fiber操作。
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
```

如果比较结果一致，就不进行后续Fiber流程。





### React.lazy做了什么？怎么做的？

它能让你像渲染常规组件一样处理动态引入的组件，配合 webpack 的 Code Splitting ，只有当组件被加载，对应的资源才会导入 ，从而达到懒加载的效果。

``` js
// 不使用 React.lazy
import OtherComponent from './OtherComponent';
// 使用 React.lazy
const OtherComponent = React.lazy(() => import('./OtherComponent'))
```

React.lazy 需要配合 Suspense 组件一起使用，在 Suspense 组件中渲染 React.lazy 异步加载的组件。如果单独使用 React.lazy，React 会给出错误提示。


React.lazy不支持服务端渲染，使用服务端渲染的同学，请绕行至 react-loadable和 loadable-components。

从实现上来看，React.lazy其实是给组件增加了标识位和回调函数[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react/src/ReactLazy.js#L104)。

``` js
//React.Lazy实现
export function lazy<T>(
  ctor: () => Thenable<{default: T, ...}>,
): LazyComponent<T, Payload<T>> {
  const payload: Payload<T> = {
    // We use these fields to store the result.
    //用来标记
    _status: -1,
    _result: ctor,
  };
  //增加标识位
  const lazyType: LazyComponent<T, Payload<T>> = {
    $$typeof: REACT_LAZY_TYPE,
    _payload: payload,
    _init: lazyInitializer,
  };
}
```


然后在fiber的开始阶段，而不是整个项目初始化的阶段。再去执行传入的thenable函数[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1278)

``` js
    //如果是lazy组件的话，走单独的流程
    case LazyComponent: {
      const elementType = workInProgress.elementType;
      return mountLazyComponent(
        current,
        workInProgress,
        elementType,
        updateLanes,
        renderLanes,
      );
    }
```



# [React](https://reactjs.org/) &middot; [![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/facebook/react/blob/master/LICENSE) [![npm version](https://img.shields.io/npm/v/react.svg?style=flat)](https://www.npmjs.com/package/react) [![CircleCI Status](https://circleci.com/gh/facebook/react.svg?style=shield&circle-token=:circle-token)](https://circleci.com/gh/facebook/react) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://reactjs.org/docs/how-to-contribute.html#your-first-pull-request)

React is a JavaScript library for building user interfaces.

* **Declarative:** React makes it painless to create interactive UIs. Design simple views for each state in your application, and React will efficiently update and render just the right components when your data changes. Declarative views make your code more predictable, simpler to understand, and easier to debug.
* **Component-Based:** Build encapsulated components that manage their own state, then compose them to make complex UIs. Since component logic is written in JavaScript instead of templates, you can easily pass rich data through your app and keep state out of the DOM.
* **Learn Once, Write Anywhere:** We don't make assumptions about the rest of your technology stack, so you can develop new features in React without rewriting existing code. React can also render on the server using Node and power mobile apps using [React Native](https://reactnative.dev/).

[Learn how to use React in your own project](https://reactjs.org/docs/getting-started.html).

## Installation

React has been designed for gradual adoption from the start, and **you can use as little or as much React as you need**:

* Use [Online Playgrounds](https://reactjs.org/docs/getting-started.html#online-playgrounds) to get a taste of React.
* [Add React to a Website](https://reactjs.org/docs/add-react-to-a-website.html) as a `<script>` tag in one minute.
* [Create a New React App](https://reactjs.org/docs/create-a-new-react-app.html) if you're looking for a powerful JavaScript toolchain.

You can use React as a `<script>` tag from a [CDN](https://reactjs.org/docs/cdn-links.html), or as a `react` package on [npm](https://www.npmjs.com/package/react).

## Documentation

You can find the React documentation [on the website](https://reactjs.org/docs).  

Check out the [Getting Started](https://reactjs.org/docs/getting-started.html) page for a quick overview.

The documentation is divided into several sections:

* [Tutorial](https://reactjs.org/tutorial/tutorial.html)
* [Main Concepts](https://reactjs.org/docs/hello-world.html)
* [Advanced Guides](https://reactjs.org/docs/jsx-in-depth.html)
* [API Reference](https://reactjs.org/docs/react-api.html)
* [Where to Get Support](https://reactjs.org/community/support.html)
* [Contributing Guide](https://reactjs.org/docs/how-to-contribute.html)

You can improve it by sending pull requests to [this repository](https://github.com/reactjs/reactjs.org).

## Examples

We have several examples [on the website](https://reactjs.org/). Here is the first one to get you started:

```jsx
function HelloMessage({ name }) {
  return <div>Hello {name}</div>;
}

ReactDOM.render(
  <HelloMessage name="Taylor" />,
  document.getElementById('container')
);
```

This example will render "Hello Taylor" into a container on the page.

You'll notice that we used an HTML-like syntax; [we call it JSX](https://reactjs.org/docs/introducing-jsx.html). JSX is not required to use React, but it makes code more readable, and writing it feels like writing HTML. If you're using React as a `<script>` tag, read [this section](https://reactjs.org/docs/add-react-to-a-website.html#optional-try-react-with-jsx) on integrating JSX; otherwise, the [recommended JavaScript toolchains](https://reactjs.org/docs/create-a-new-react-app.html) handle it automatically.

## Contributing

The main purpose of this repository is to continue evolving React core, making it faster and easier to use. Development of React happens in the open on GitHub, and we are grateful to the community for contributing bugfixes and improvements. Read below to learn how you can take part in improving React.

### [Code of Conduct](https://code.fb.com/codeofconduct)

Facebook has adopted a Code of Conduct that we expect project participants to adhere to. Please read [the full text](https://code.fb.com/codeofconduct) so that you can understand what actions will and will not be tolerated.

### [Contributing Guide](https://reactjs.org/contributing/how-to-contribute.html)

Read our [contributing guide](https://reactjs.org/contributing/how-to-contribute.html) to learn about our development process, how to propose bugfixes and improvements, and how to build and test your changes to React.

### Good First Issues

To help you get your feet wet and get you familiar with our contribution process, we have a list of [good first issues](https://github.com/facebook/react/labels/good%20first%20issue) that contain bugs which have a relatively limited scope. This is a great place to get started.

### License

React is [MIT licensed](./LICENSE).
