---
title: "React global state by Context API"
date: 2018-10-05T12:00:00+09:00
tags: ["react", "context", "global-state"]
---

##### A tiny library to help storing global state in React.

I had been trying to find a way to use React without using class. Redux is one solution to achieve this. Although I love the idea of writing everthing in pure functions, Redux is sometimes not suitable for small apps. React v16.3 introduced new Context API officially. Since then, several ideas were proposed to use it for managing global state. So far, I wasn't able to find something I really liked, hence I made a new one.

### The example code

The following code is a working example. Hope it shows the idea well.

```jsx
import React from 'react';
import ReactDOM from 'react-dom';

import { createGlobalState } from 'react-context-global-state';

const initialState = {
  counter1: 0,
};
const { StateProvider, StateConsumer } = createGlobalState(initialState);

const Counter = () => (
  <StateConsumer name="counter1">
    {(value, update) => (
      <div>
        <span>Count: {value}</span>
        <button onClick={() => update(v => v + 1)}>+1</button>
      </div>
    )}
  </StateConsumer>
);

const App = () => (
  <StateProvider>
    <h1>Counter</h1>
    <Counter />
    <Counter />
  </StateProvider>
);

ReactDOM.render(<App />, document.getElementById('app'));
```

### Technical notes

The `initialState` is an object. For each properties of the object, a context is created. `StateProvider` wraps all the context providers and `StateConsumer` requires a `name` prop to specify which context consumer to use.

### The library package

There is a package to support the above example. You can install it:

```bash
npm install react-context-global-state
```

You can also check out the source code:

https://github.com/dai-shi/react-context-global-state

### Summary

In this short post, I introduced a way to use global state with React Context API. A library package is developed and example code works with it. Hope anyone interested would try it out, and drop me a note. Thanks.
