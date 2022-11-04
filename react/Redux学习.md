### 前言
在react中，页面以**组件**为基本单位，通过组件的排列组合、嵌套构成页面。在嵌套组件之间可以通过`props`传递状态（`state`），而在**非嵌套组件**（并列组件）之间,虽说也有pub-sub或context作为通信工具，但是对于数据状态非常复杂的应用，这些通信方式很难保证state的数据安全性。
这个时候，迫切需要一个机制，**把所有的state集中到一起管理，能够灵活的将所有state各取所需的分发给所有的组件**，这就是redux。

### 简介
-   redux是的诞生是为了给 React 应用提供「可预测化的状态管理」机制。
-   Redux会将整个应用状态(其实也就是数据)存储到到一个地方，称为store
-   这个store里面保存一棵状态树(state tree)
-   组件改变state的唯一方法是通过调用store的dispatch方法，触发一个action，这个action被对应的reducer处理，于是state完成更新
-   组件可以派发(dispatch)行为(action)给store,而不是直接通知其它组件
-   其它组件可以通过订阅store中的状态(state)来刷新自己的视图

![[Pasted image 20221104173049.png]]
### redux基本使用
#### 1、创建store
```js
//store.js
//原`createStore`被官方替换为`legacy_createStore`
import { legacy_createStore, applyMiddleware } from 'redux'
// 引入thunk支持异步方法
import thunk from 'redux-thunk'
//引入reducer
import countReducer from './countReducer'
//根据reducer创建store
const store = legacy_createStore(countReducer,applyMiddleware(thunk))
export store
```
在`store`中，需要引入对应的`reducer`，一个reducer对应一个组件。目前演示暂时仅创建一个reducer，下文中会介绍`reducer合并`。

#### 2、创建reducer
```js
//countReducer.js

//初始化state
const initState = 0

const countReducer = (preState: number = initState, action) => {
  const { type, data } = action
  switch (type) {
    case 'increment':       //根据对应的type执行action
      return preState + data
     case 'asyncIncrement'：
     return preState + data
    default:
      return preState
  }
}
export default countReducer
```
- 在reducer中初始化state
- action发出命令后将state放入reucer加工函数中，返回新的state,对state进行加工处理
#### 3、创建action
```js
//countAction.js
export const increment = (data: number) => ({ type: 'increment', data })

//异步方法
export const asyncIncrement = (data: number, time: any) => {
  return (dispatch: any) => {
    setTimeout(() => {
      dispatch(increment(data))
    }, time)
  }
}
```
-   这个action可以理解为指令，需要发出多少动作就有多少指令
-   action是一个对象，必须有一个叫type的参数，定义action类型

这里用一个例子简单演示一下如何使用：
``` jsx
//count.js

//引入store
import store from '../../store'
//引入action
import { increment,asyncIncrement } from '../../store/actions/countAction'

function CountNumber() {
  //通过store.getState()获取state中的初始值
  const count = store.getState()  
  
  const add = () => {
  //通过store.dispatch将action发送到reducer进行状态更新
    store.dispatch(increment(1))
  }
  
  //异步增加
  const asyncAdd = () => {
    store.dispatch(asyncIncrement(1))
  }
  return (
    <div className="App">
      <div>当前计数总和为：{count}</div>
      <br />
      <button onClick={add}>+1</button>
      <button onClick={asyncAdd}>+1</button>
    </div>
  )
}
export default CountNumber
```
**这里需要注意的是**，因为store创建的**state并非是react响应对象**，要通过`store.subscribe`订阅组件之后，检测redux中状态，如果状态发生变化就重新调用render：
```jsx
//index.jsx
import store from './store'

const root = ReactDOM.createRoot(document.getElementById('root') as HTMLElement)

root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)

//对根组件进行订阅
store.subscribe(() => {
  root.render(
    <React.StrictMode>
      <App />
    </React.StrictMode>
  )
})
```

### react-redux
react-redux是官方提供的用于绑定redux的库，**使用react-redux后将不在通过store.subscribe订阅，自动更新状态。**
#### 1、基本理念
- react-redux将组件分成**UI组件**和**容器组件**
- 容器组件包裹所有的UI组件
- 容器组件用于跟redux交互通信
- UI组件中不能使用任何redux的api
- 容器组件会给UI组件传递：（1）redux中所有保存的状态（2）用于操作状态的方法
- 容器组件给UI传递的状态、操作方法均通过`Props`传递
![[Pasted image 20221104185514.png]]
#### 2、核心
###### (1).Provider
通过`Provider`将`store`分发给容器组件，一般我们都将顶层组件包裹在Provider组件之中，这样的话，所有组件就都可以在react-redux的控制之下了：
```jsx
<Provider store = {store}>
  <App /> 
<Provider>
```
###### (2).connect
react-redux通过`connect(mapStateToProps, mapDispatchToProps)(MyComponent)`函数将组件变成容器组件。
这里用一个例子简单演示一下如何使用：
```jsx
import { connect } from 'react-redux'
import { increment,asyncIncrement } from '../../store/actions/countAction'

//这部分为ui组件，通过props获取store中的state和方法
function CountNumber(props) {
  const add = () => {
    props.increment(1)    
  }
  return (
    <div className="App">
      <div>当前计数总和为：{props.count}</div>    
      <br />
      <button onClick={add}>+1</button>
    </div>
  )
}
/**
 * mapStateToProps函数返回的对象中的key作为传递给UI组件props的key，value就作为传递给UI组件props的value--状态
 */
const mapStateToProps = (state) => ({ count: state })
/**
 *
 * mapDispatchToProps函数返回的对象中的key就作为传递给UI组件props的key，value就作为传递给UI组件props的value--操作方法
 */
const mapDispatchToProps = (dispatch) => {
  return {
    increment: (num: number) => {
      dispatch(increment(num))
    },
    asyncIncrement:(num) => {
      dispatch(asyncIncrement(num))
    }
  }
}
/*
  mapDispatchToProps简写,直接写成对象key为props的key，value为action中的方法
  const mapDispatchToProps = {increment,asyncIncrement}
*/

//用`connect(mapStateToProps, mapDispatchToProps)(MyComponent)`将UI组件变成容器组件后导出
export default connect(mapStateToProps, mapDispatchToProps)(CountNumber)
```

### 合并reducer
如果有多个reducer，可以通过`combineReducers()`合并：
```js
import { legacy_createStore } from 'redux'
//引入count组件的reducer
import countReducer from './countReducer'
//引入person组件的reducer
import personReducer from './personReducer'
//合并reducer
const allRedecer = combineReducers({
  count:countReducer,
  person:personReducer
})
//根据reducer创建store
const store = legacy_createStore(allRedecer)
export store
```
合并之后的，所有reducer中的state都汇总到一个state中：
```js
//汇总后的state
state {
  count:{},    //countReducer创建的state
  person:{}    //personReducer创建的state
}
```
因此在使用的时候要注意state的层级。

### Redux Toolkit（redux最佳实践）
