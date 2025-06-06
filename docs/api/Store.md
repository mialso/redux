---
id: store
title: Store
description: 'API > Store: the core Redux store methods'
---

# Store

A store holds the whole [state tree](../understanding/thinking-in-redux/Glossary.md#state) of your application.
The only way to change the state inside it is to dispatch an [action](../understanding/thinking-in-redux/Glossary.md#action) on it, which triggers the [root reducer function](../understanding/thinking-in-redux/Glossary.md#reducer) to calculate the new state.

A store is not a class. It's just an object with a few methods on it.

To create a store, **pass your root [reducer function](../understanding/thinking-in-redux/Glossary.md#reducer) to Redux Toolkit's [`configureStore` method](https://redux-toolkit.js.org/api/configureStore)**, which will set up a Redux store with a good default configuration. (Alternately, if you're not yet using Redux Toolkit, you can use the original [`createStore`](createStore.md) method, but we encourage you to [migrate your code to use Redux Toolkit](../usage/migrating-to-modern-redux.mdx) as soon as possible)

## Store Methods

### getState()

Returns the current state tree of your application.
It is equal to the last value returned by the store's reducer.

#### Returns

_(any)_: The current state tree of your application.

---

&nbsp;

### dispatch(action)

Dispatches an action. This is the only way to trigger a state change.

The store's reducer function will be called with the current [`getState()`](#getstate) result and the given `action` synchronously. Its return value will be considered the next state. It will be returned from [`getState()`](#getstate) from now on, and the change listeners will immediately be notified.

:::caution

If you attempt to call `dispatch` from inside the [reducer](../understanding/thinking-in-redux/Glossary.md#reducer), it will throw with an error saying "Reducers may not dispatch actions." Reducers are pure functions - they can _only_ return a new state value and must not have side effects (and dispatching is a side effect).

In Redux, subscriptions are called after the root reducer has returned the new state, so you _may_ dispatch in the subscription listeners. You are only disallowed to dispatch inside the reducers because they must have no side effects. If you want to cause a side effect in response to an action, the right place to do this is in the potentially async [action creator](../understanding/thinking-in-redux/Glossary.md#action-creator).

:::

#### Arguments

1. `action` (_Object_<sup>†</sup>): A plain object describing the change that makes sense for your application. Actions are the only way to get data into the store, so any data, whether from the UI events, network callbacks, or other sources such as WebSockets needs to eventually be dispatched as actions. Actions must have a `type` field that indicates the type of action being performed. Types can be defined as constants and imported from another module. It's better to use strings for `type` than [Symbols](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Symbol) because strings are serializable. Other than `type`, the structure of an action object is really up to you. If you're interested, check out [Flux Standard Action](https://github.com/acdlite/flux-standard-action) for recommendations on how actions could be constructed.

#### Returns

(Object<sup>†</sup>): The dispatched action (see notes).

#### Notes

<sup>†</sup> The “vanilla” store implementation you get by calling [`createStore`](createStore.md) only supports plain object actions and hands them immediately to the reducer.

However, if you wrap [`createStore`](createStore.md) with [`applyMiddleware`](applyMiddleware.md), the middleware can interpret actions differently, and provide support for dispatching [async actions](../understanding/thinking-in-redux/Glossary.md#async-action). Async actions are usually asynchronous primitives like Promises, Observables, or thunks.

Middleware is created by the community and does not ship with Redux by default. You need to explicitly install packages like [redux-thunk](https://github.com/reduxjs/redux-thunk) or [redux-promise](https://github.com/acdlite/redux-promise) to use it. You may also create your own middleware.

To learn how to describe asynchronous API calls, read the current state inside action creators, perform side effects, or chain them to execute in a sequence, see the examples for [`applyMiddleware`](applyMiddleware.md).

#### Example

```js
import { createStore } from 'redux'
const store = createStore(todos, ['Use Redux'])

function addTodo(text) {
  return {
    type: 'ADD_TODO',
    text
  }
}

store.dispatch(addTodo('Read the docs'))
store.dispatch(addTodo('Read about the middleware'))
```

---

&nbsp;

### subscribe(listener)

Adds a change listener. It will be called any time an action is dispatched, and some part of the state tree may potentially have changed. You may then call [`getState()`](#getstate) to read the current state tree inside the callback.

You may call [`dispatch()`](#dispatchaction) from a change listener, with the following caveats:

1. The listener should only call [`dispatch()`](#dispatchaction) either in response to user actions or under specific conditions (e. g. dispatching an action when the store has a specific field). Calling [`dispatch()`](#dispatchaction) without any conditions is technically possible, however it leads to an infinite loop as every [`dispatch()`](#dispatchaction) call usually triggers the listener again.

2. The subscriptions are snapshotted just before every [`dispatch()`](#dispatchaction) call. If you subscribe or unsubscribe while the listeners are being invoked, this will not have any effect on the [`dispatch()`](#dispatchaction) that is currently in progress. However, the next [`dispatch()`](#dispatchaction) call, whether nested or not, will use a more recent snapshot of the subscription list.

3. The listener should not expect to see all state changes, as the state might have been updated multiple times during a nested [`dispatch()`](#dispatchaction) before the listener is called. It is, however, guaranteed that all subscribers registered before the [`dispatch()`](#dispatchaction) started will be called with the latest state by the time it exits.

It is a low-level API. Most likely, instead of using it directly, you'll use React (or other) bindings. If you commonly use the callback as a hook to react to state changes, you might want to [write a custom `observeStore` utility](https://github.com/reduxjs/redux/issues/303#issuecomment-125184409). The `Store` is also an [`Observable`](https://github.com/zenparsing/es-observable), so you can `subscribe` to changes with libraries like [RxJS](https://github.com/ReactiveX/RxJS).

To unsubscribe the change listener, invoke the function returned by `subscribe`.

#### Arguments

1. `listener` (_Function_): The callback to be invoked any time an action has been dispatched, and the state tree might have changed. You may call [`getState()`](#getstate) inside this callback to read the current state tree. It is reasonable to expect that the store's reducer is a pure function, so you may compare references to some deep path in the state tree to learn whether its value has changed.

##### Returns

(_Function_): A function that unsubscribes the change listener.

##### Example

```js
function select(state) {
  return state.some.deep.property
}

let currentValue
function handleChange() {
  let previousValue = currentValue
  currentValue = select(store.getState())

  if (previousValue !== currentValue) {
    console.log(
      'Some deep nested property changed from',
      previousValue,
      'to',
      currentValue
    )
  }
}

const unsubscribe = store.subscribe(handleChange)
unsubscribe()
```

---

&nbsp;

### replaceReducer(nextReducer)

Replaces the reducer currently used by the store to calculate the state.

It is an advanced API. You might need this if your app implements code splitting, and you want to load some of the reducers dynamically. You might also need this if you implement a hot reloading mechanism for Redux.

#### Arguments

1. `nextReducer` (_Function_) The next reducer for the store to use.
