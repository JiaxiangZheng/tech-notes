要问 2015 年最火的前端开发技术栈，那就不得不提及 React + Redux + Webpack 了。React 是 FB 提出的新一代组件式开发框架，极大简化了日益复杂的前端业务开发，但对于整个页面的数据状态控制，则不在 React 范围内。FB 本身提出了一个单向数据流 Flux 编程范式，将整个数据流程加以简化，避免出现数据的不可控。

由于 Flux 只是一种编程的思想，因此开源社区中出现了[多种实现的方案](https://medium.com/@dan_abramov/the-evolution-of-flux-frameworks-6c16ad26bb31#.t9257ihh5)，而 FB 自家也实现了一套基于事件的 Flux 方案。然而 Flux 官方代码在使用上，还是会显得不那么好用，如异步问题、同构渲染，于是乎出现了诸多实现对其改进。而 Redux 就是改进实现中的佼佼者。

Redux 本身的源码非常地短小精悍，估计加在一起也就不到一千行，非常适合仔细研读一番。它并没有与任何框架耦合，这意味着它可以和任何框架组合到一起。Redux 源码的实现质量非常高，可谓大呼痛快。这里简单地对源码作一定的解读。

先解释一下 Redux 中提出的几个概念：

1. action：用于将数据传递到 store 的载体，是数据更新的来源，一般通过 `dispatch(action)` 的方式更新整个应用的状态；一般都有 action 创建函数，返回带数据和 action type 的对象，供 reducer 更新应用状态；
2. reducer：接收上一个状态和 action 作为输入，返回的是更新后的 state（注意，Redux 推崇无侵入的 reducer 尽可能降低其带来的副作用影响，即 reducer 中不在原 state 上直接作更改，，它只是重新返回一个新的对象）
2. store：通过 Redux 的 createStore 方法创建，作为全局数据状态的管理中心，一方面它存储着全局的状态数据；另一方面，它提供订阅发布的功能，以便注册及触发全局状态监听回调。

### 基本结构

Redux 要求氢整个页面的数据状态当作一个对象统一进行管理，通过 createStore 返回出数据管理 store，即：

```javascript
function createStore(reducer, initState) {
	// 向 store 注册监听回调，每次 dispatch 调用都会触发所有回调调用
	subscribe = (fn) => {
	}
	// 触发一个 action，调用 reducer 处理该 action，以更新整个 state；
	// Redux 中假定每个 action 都是一个纯的对象（plain object），并包含 type 属性
	dispatch = (action) => {
	}
	return {
		dispatch, subscribe, ...
	}
}
```

好了，一个简单的 Redux 框架就已经分析完了。我们行文到此。。。使用这个简单的接口我们就能开工干活了：

```javascript
function setName(name) {
    return {
        type: 'SET_NAME',
        name: name
    }
}

function reducer(state = {name: ''}, action) {
    switch (action.type) {
        case 'SET_NAME':
            return Object.assign({}, state, {name: action.name});
        default :
            return state;
    }
}

let store = redux.createStore(reducer);
store.subscribe(function () {
    console.log(store.getState());
});

store.dispatch(setName('Jiaxiang Zheng'));
```

### combineReducers

不，作为一个已经拥有生态圈的技术，要这就完了，岂不是有种『我裤子都脱了，你给我看这个』的感觉？

从上面的代码片段中，不难看出，reducer扮演的角色是将 action 真正地进行处理，并生成新的 state 返回出来。但是，当一个页面复杂到一定程度时，显然我们无法像上面这个 reducer 这样，直接把所有的改动行为都封装在一个函数中。这会导致 switch 分支庞大难以管理。事实上，reducer 之所以被称为 reducer，与函数式编程中的 map-reduce 也是有一定关系的。Redux 的 combineReducers 允许我们将 reducer 进行细化拆分，最后合并成一个总的汇总函数传递到 crateStore 中（将 state 各个部分进行汇总，即所谓的 reduce 概念），这么来看，对于 crateStore，其参数并没有任何实质的变化。我们看看 combineReducers 是怎么实现的吧！

除去参数检查的代码，`combineReducers` 实现非常简短（参数检查代码这里不表）：

```javascript
export default function combineReducers(reducers) {
  // 1. 检查输入的参数，去除非法的 reducers，假设为 finalReducers 变量

  // 2. 先获取各 state 的默认值
  var defaultState = mapValues(finalReducers, () => undefined)

  return (state = defaultState, action) => {
    var hasChanged = false
    // 由于 reducers 本身是一个 key-value 形式的对象
    // key 为 state 中对应的 key，而 value 即对应的 reducer
    var finalState = mapValues(finalReducers, (reducer, key) => {
      var previousStateForKey = state[key]
      var nextStateForKey = reducer(previousStateForKey, action)
	  
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
      return nextStateForKey
    })

    return hasChanged ? finalState : state
  }
}
```

这里，提一下上面用到的 `mapValues` 辅助函数，它对对象中每一个 value（key 所对应的值），调用指定的回调函数 fn，并返回最终的 `<key, fn(value)>` 键值对。

```
export default function mapValues(obj, fn) {
  return Object.keys(obj).reduce((result, key) => {
    result[key] = fn(obj[key], key)
    return result
  }, {})
}
```

### applyMiddleware

我们都知道 Redux 非常地精简，底层并没有作过多额外的事情，那如果我们想要在它每次状态改变的事情做点额外的事情（比如日志记录等），亦或我们希望执行异步的 action 呢？显然从上面的代码来看，它并不支持。

但是，Redux 提供了一个 applyMiddleware 以无侵入的方式注入我们自己的代码，这里称之为中间件。正常情况下我们创建 store 用 `createStore(rootReducer, initState)` 这种方式，但是有了中间件，我们可以以：

```let store = applyMiddleware(middleware)(createStore)(rootReducer, initState)```

的方式创建 store 了。它对原来的 createStore 进行了一次包装，将我们自定义的中间件代码包含在其中。我们来看看这个神奇的 applyMiddleware 是如何实现的吧？

```javascript
export default function applyMiddleware(...middlewares) {
  // 返回的实际上是一个高阶函数，其中的 next 即我们的 createStore
  return (next) => (reducer, initialState) => {
  	 // 首先生成最原始的 store，并保存其对应的初始 dispatch 方法
    var store = next(reducer, initialState)
    var dispatch = store.dispatch
    var chain = []

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    // 自定义的中间件接收一个对象 {getState, dispatch} 作为其参数，返回一个函数
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    
    // compose(f, g, h) 即等价于 arg => f(g(h(arg)))，实际上是一个函数，接收 dispatch 作为参数，返回新的 dispatch 函数
    //   var fn = h(dispatch);
    //   fn = g(res); 
    //   fn = f(res);   
    // 最终的 fn 即新的 dispatch 函数
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

为了更好地了解自定义中间件，看下 redux-thunk 是怎么做的吧！

```
function thunkMiddleware({ dispatch, getState }) {
  return next => action =>
    typeof action === 'function' ?
      action(dispatch, getState) :
      next(action);
}

module.exports = thunkMiddleware
```

类似于 Koa，将上一个处理函数当作输入，中间件自身处理完之后，就将 action 扔给下一个去处理，当然了，redux-thunk 为支持异步，允许 action 为函数，此时将会将它延迟执行而继续下一个 action。

### bindActionCreators

按照上面 Redux 的工作流程，最后的触发无非就是要调用 `dispatch(setAction(value))` 这种形式通知 reducer 去改变 state，但这样的话，每个触发的地方都得知道 dispatch 这个函数。比如，React 的组件式开发中，就会出现不断地把 dispatch 方法通过根组件一级一级向下传播的行为。这种代码会显得过于冗长，而 bindActionCreators 可以简化这个问题：

```
<Root props={...bindActionCreators({
  setTitle: actions.setTitle,
  fetchList: actions.fetchList
})}></Root>
```

这样，在 `Root` 组件中，直接使用 `this.props.setTitle(title)` 即可触发对应的 action，进而引起 reducer 中的变化，导致整个应用重新渲染。

Redux 中的实现也很简单：

```
function bindActionCreator(actionCreator, dispatch) {
  return (...args) => dispatch(actionCreator(...args))
}

function bindActionCreators(actionCreators, dispatch) {
  // 还记得前面提到的 mapValues 函数吗？
  return mapValues(actionCreators, actionCreator =>
    bindActionCreator(actionCreator, dispatch)
  )
}
```
***

最后，这里有一篇[对 Redux 作者的采访](http://survivejs.com/blog/redux-interview/)，非常有意思，值得一读。另外，在 stackoverflow 上有一个[帖子](http://stackoverflow.com/questions/32461229/why-use-redux-over-facebook-flux)关于 Redux 为何优于 Flux，Redux 作者 Dan Abramov 给出的回答也值得思考。
