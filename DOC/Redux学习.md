# Redux 简单回顾

Redux 是 JavaScript 状态容器。提供可预测化得状态管理

更新 state 的数据，action 改变。

## Redux的三大核心概念

- Action
- Reducer
- Store

其中**Action**理解为**动作**，**store**可以理解为**存储**

Action 表示应用中的各类动作或操作，不同的操作会改变应用相应的state状态，既可以表示为带 `type` 的对象

Store 则是我们存储 state 的地方，通过 redux 当中的 createStore 方法来创建一个 store。

```js
const createStore = (reducer) => {
    let state;
    let listeners = [];
    // 用来返回当前的state
    const getState = () => state;
    // 根据action调用reducer返回新的state并触发listener
    const dispatch = (action) => {
        state = reducer(state, action);
        listeners.forEach(listener => listener());
    };

    const subscribe = (listener) => { //订阅
        listeners.push(listener);
        return () => {
            listeners = listeners.filter(l => l !== listener);
        };
    };
    return { getState, dispatch, subscribe };
}
```

>为什么被称为reducer?

官方文档：It's called a reducer because it's the type of function you would pass to Array.prototype.reduce(reducer, ?initialValue)

```js
const sum =[0, 1, 2, 3].reduce( (prev, curr) => prev + curr );
console.log(sum); // 6
```

是不是很类似于 reducer 会接受 state 和 action 并返回新的state

```js
const todos = (state = [], action) => {
    switch (action.type) {
    case 'ADD_TODO':
        return [
            ...state,
            {
                id: action.id,
                text: action.text,
                completed: false
            }
        ];
    default:
        return state;
    }
}
```

reducer - 函数式编程 - Fold折叠 - Array.prototype.reduce 回调 - 缩减

一个简单的state例子：

## Redux 三大原则

- 单一数据源
- State只读
- 纯函数修改

1. 整个应用的 state 被储存在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中。
2. 唯一改变 state 的方法就是触发 action，action 是一个用于描述已发生事件的普通对象
3. 为了描述 action 如何改变 state tree ，你需要编写 reducers

