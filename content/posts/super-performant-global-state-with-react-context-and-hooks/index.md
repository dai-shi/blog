---
title: "Super performant global state with React context and hooks"
subtitle: "Yet another Redux-like library"
date: 2019-06-15T23:00:00+09:00
tags: ["react", "context", "hooks", "global-state", "proxy"]
---

### Introduction

There are many libraries to provide global state in React.
React itself doesn't provide such a feature,
probably because separation of concerns is important
and having global state naively is not idiomatic.
However, in certain cases, having global state is good
as long as it's properly implemented.
It's likely that performance drops down
compared to using non-global state (incl. multiple contexts).

This post introduces a library for global state with performance.

### Problem

Combining context and useReducer and develop
a Redux-like feature is easy.
One would say it's enough if they don't
need Redux DevTools and Redux middleware.

But still, there is a problem if an app gets bigger.
Technically, useContext doesn't have mechanism to bail out,
and all components that useContext re-render every time
context value is changed.
That is why react-redux gave up using context directly
and move back to subscriptions.

Anyhow, this problem happens if you use context value
for a single big state object. Unless your app
is very small, this limitation can't be ignored.

Another problem is how to specify which part of state
a component needs to render. Selectors are often
used in such a scenario, but it is not trivial to write
proper selectors unless you have good knowledge of
referential equality and memoization.

### Solution

The first problem is solved by stopping context propagation
when context value is changed. This is done by undocumented
feature called "calculateChangedBits". Because propagation is
stopped, no updates are pushed to components, and now components
need to pull changes. We use subscriptions for that.
Some experienced developers might think why we still need to
use context if we use subscriptions. This is an assumption,
but using context is safer for concurrent mode and probably
fits better for React developer tools.

The second problem is solved by tracking state usage in component
rendering. This is done by Proxy.
It's a bit magical, but basically it's only for performance
optimization. It doesn't change the semantics at all.

### Library

I implemented these features as a library.

https://github.com/dai-shi/react-tracked

It's still new as of writing, but it's ready for review.

### Example

```jsx
import React, { useReducer } from 'react';
import ReactDOM from 'react-dom';

import { Provider, useTracked } from 'react-tracked';

const initialState = {
  counter: 0,
  text: 'hello',
};

const reducer = (state, action) => {
  switch (action.type) {
    case 'increment': return { ...state, counter: state.counter + 1 };
    case 'decrement': return { ...state, counter: state.counter - 1 };
    case 'setText': return { ...state, text: action.text };
    default: throw new Error(`unknown action type: ${action.type}`);
  }
};

const useValue = () => useReducer(reducer, initialState);

const Counter = () => {
  const [state, dispatch] = useTracked();
  return (
    <div>
      {Math.random()}
      <div>
        <span>Count:{state.counter}</span>
        <button type="button" onClick={() => dispatch({ type: 'increment' })}>+1</button>
        <button type="button" onClick={() => dispatch({ type: 'decrement' })}>-1</button>
      </div>
    </div>
  );
};

const TextBox = () => {
  const [state, dispatch] = useTracked();
  return (
    <div>
      {Math.random()}
      <div>
        <span>Text:{state.text}</span>
        <input value={state.text} onChange={event => dispatch({ type: 'setText', text: event.target.value })} />
      </div>
    </div>
  );
};

const App = () => (
  <Provider useValue={useValue}>
    <h1>Counter</h1>
    <Counter />
    <Counter />
    <h1>TextBox</h1>
    <TextBox />
    <TextBox />
  </Provider>
);

ReactDOM.render(<App />, document.getElementById('app'));
```

### Demo

<iframe src="https://codesandbox.io/embed/github/dai-shi/react-tracked/tree/master/examples/01_minimal?fontsize=14" title="react-tracked-example" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

### Closing notes

I didn't explain everything about the library.
Most notably, this library is kind of a fork of
[reactive-react-redux](https://github.com/dai-shi/reactive-react-redux),
and actually the hooks API is identical, which is also similar to
[react-redux hooks](https://react-redux.js.org/api/hooks).
If you are a redux user and already convinced of
DevTools and middleware, just use those libraries.
