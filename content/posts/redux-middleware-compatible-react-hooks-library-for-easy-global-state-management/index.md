---
title: "Redux middleware compatible React Hooks library for easy global state management"
subtitle: "React hooks are awesome."
date: 2018-11-07T12:00:00+09:00
tags: ["react", "hooks", "typescript", "global-state", "redux"]
---

### The library

I've been developing a library to make use of React Hooks API. The library is for global state mangement with the aim to use it instead of Redux, especially for smaller applications. See the repository for more information and the source code.

https://github.com/dai-shi/react-hooks-global-state

Please also check out my previous posts
[[1](https://blog.axlight.com/posts/typescript-aware-react-hooks-for-global-state/)]
[[2](https://blog.axlight.com/posts/an-alternative-to-react-redux-by-react-hooks-api-for-both-javascript-and-typescript/)].

### Support for Redux middleware

Originally, the library was extending useState for global state. Later, reducer is supported. Now, middleware support is added. The following is the code snippet from the working example to illustrate how it can be used.

At first, we import `createStore` from our library and two helpers from Redux.

```javascript
import { createStore } from 'react-hooks-global-state';
import { applyMiddleware, combineReducers } from 'redux';
```

We then define the initial state. Note, the initial state is mandatory in our library, whereas it's optional in Redux.

```javascript
const initialState = {
  counter: 0,
  person: {
    age: 0,
    firstName: '',
    lastName: '',
  },
};
```

We define two reducers for each state items ( `counter` and `person` ).

```javascript
const counterReducer = (state = initialState.counter, action) => {
  switch (action.type) {
    case 'increment': return state + 1;
    case 'decrement': return state - 1;
    default: return state;
  }
};
const personReducer = (state = initialState.person, action) => {
  switch (action.type) {
    case 'setFirstName': return {
      ...state,
      firstName: action.firstName,
    };
    case 'setLastName': return {
      ...state,
      lastName: action.lastName,
    };
    case 'setAge': return {
      ...state,
      age: action.age,
    };
    default: return state;
  }
};
```

Then, combine them with `combineReducers`.

```javascript
const reducer = combineReducers({
  counter: counterReducer,
  person: personReducer,
});
```

We define logging middleware which is from the Redux documentation.

```javascript
const logger = ({ getState }) => next => (action) => {
  console.log('will dispatch', action);
  const returnValue = next(action);
  console.log('state after dispatch', getState());
  return returnValue;
};
```

We create a store based on what we defined above, and export the resulting functions.

```javascript
export const { GlobalStateProvider, dispatch, useGlobalState } = createStore(
  reducer,
  initialState,
  applyMiddleware(logger),
);
```

For example, the counter component is implemented like the following.

```jsx
const Counter = () => {
  const [value] = useGlobalState('counter');
  return (
    <div>
      <span>Count: {value}</span>
      <button
        type="button"
        onClick={() => dispatch({ type: 'increment' })}
      >
        Increment
      </button>
    </div>
  );
};
```

### The full example code in TypeScript

You can see the full example code in the [repository](https://github.com/dai-shi/react-hooks-global-state/tree/master/examples/07_middleware). It is written in TypeScript, but JavaScript developers could read it by just ignoring type definisions (`type Foo = ...;`) and type annotations (`: Foo` just after variables).

### It's working but more experiments are needed

As far as I tried, the logger middleware works as expected. I haven't tried other middleware yet. Especially, async libraries should be tried. If anybody is interested in trying some other middleware, I'd be happy to help.

### A final note

React 16.7 which will introduce the Hooks API is not yet released. Until it's official, anything that relies on hooks should be experimental. But, you've got to try it anyway.

### Changelog

- _[2018-11-07]:_ Initial publication.
- _[2018-11-08]:_ Follow the library API change and make the example more intuitive.
- _[2018-11-12]:_ The library API is changed again based on the Context API.
