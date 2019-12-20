---
title: "How to Handle Async Actions for Global State With React Hooks and Context"
subtitle: "With React Tracked"
tags: ["react", "context", "hooks", "global-state", "async"]
date: 2019-12-20T23:00:00+09:00
---

### Introduction

I have been developing React Tracked, which is
a library for global state with React Hooks and Context.

<https://react-tracked.js.org>

This is a small library and focuses on only one thing.
It optimizes re-renders using state usage tracking.
More technically, it uses Proxies to detect the usage in render,
and only triggers re-renders if necessary.

Because of that, the usage of React Tracked is very straightforward.
It is just like the normal useContext. Here's an example.

```jsx
const Counter = () => {
  const [state, setState] = useTracked();
  // The above line is almost like the following.
  // const [state, setState] = useContext(Context);
  const increment = () => {
    setState(prev => ({ ...prev, count: prev.count + 1 }));
  };
  return (
    <div>
      {state.count}
      <button onClick={increment}>+1</button>
    </div>
  );
};
```

For a concrete example, please check out "Getting Started" in
[the doc](https://react-tracked.js.org).

Now, because React Tracked is a wrapper around React Hooks and Context,
it doesn't support async actions natively.
This post shows some examples how to handle async actions.
It's written for React Tracked, but it can be used without React Tracked.

The example we use is a simple data fetching from a server.
The first pattern is without any libraries, and uses custom hooks.
The rest is using three libraries, one of which is my own.

### Custom hooks without libraries

Let's look at a native solution. We define a store at first.

```js
import { createContainer } from 'react-tracked';

const useValue = () => useState({ loading: false, data: null });
const { Provider, useTracked } = createContainer(useValue);
```

This is one of the patterns to create a store (container) in React Tracked.
Please check out the [recipes](https://react-tracked.js.org/docs/recipes#recipes-for-createcontainer) for other patterns.

Next, we create a custom hook.

```js
const useData = () => {
  const [state, setState] = useTracked();
  const actions = {
    fetch: async (id) => {
      setState(prev => ({ ...prev, loading: true }));
      const response = await fetch(`https://reqres.in/api/users/${id}?delay=1`);
      const data = await response.json();
      setState(prev => ({ ...prev, loading: false, data }));
    },
  };
  return [state, actions];
};
```

This is a new hook based on useTracked and it returns state and actions.
You can invoke `action.fetch(1)` to start fetching.

Note: Consider wrapping with useCallback if you need a stable async function.

React Tracked actually accepts a custom hook,
so this custom hook can be embedded in the container.

```js
import { createContainer } from 'react-tracked';

const useValue = () => {
  const [state, setState] = useState({ loading: false, data: null });
  const actions = {
    fetch: async (id) => {
      setState(prev => ({ ...prev, loading: true }));
      const response = await fetch(`https://reqres.in/api/users/${id}?delay=1`);
      const data = await response.json();
      setState(prev => ({ ...prev, loading: false, data }));
    },
  };
  return [state, actions];
};

const { Provider, useTracked } = createContainer(useValue);
```

Try the working example.

<https://codesandbox.io/s/hungry-nightingale-qjeis>

### useThunkReducer

[react-hooks-thunk-reducer](https://github.com/nathanbuchar/react-hook-thunk-reducer) provides a custom hook `useThunkReducer`.
This hook returns `dispatch` which accepts a thunk function.

The same example can be implemented like this.

```js
import { createContainer } from 'react-tracked';
import useThunkReducer from 'react-hook-thunk-reducer';

const initialState = { loading: false, data: null };
const reducer = (state, action) => {
  if (action.type === 'FETCH_STARTED') {
    return { ...state, loading: true };
  } else if (action.type === 'FETCH_FINISHED') {
    return { ...state, loading: false, data: action.data };
  } else {
    return state;
  }
};

const useValue = () => useThunkReducer(reducer, initialState);
const { Provider, useTracked } = createContainer(useValue);
```

Invoking an async action would be like this.

```js
const fetchData = id => async (dispatch, getState) => {
  dispatch({ type: 'FETCH_STARTED' });
  const response = await fetch(`https://reqres.in/api/users/${id}?delay=1`);
  const data = await response.json();
  dispatch({ type: 'FETCH_FINISHED', data });
};

dispatch(fetchData(1));
```

It should be familiar to redux-thunk users.

Try the working example.

<https://codesandbox.io/s/crimson-currying-og54c>

### useSageReducer

[use-saga-reducer](https://github.com/azmenak/use-saga-reducer) provides a custom hook `useSageReducer`.
Because this library uses [External API](https://redux-saga.js.org/docs/api/index.html#external-api), you can use redux-saga without Redux.

Let's implement the same example again with Sagas.

```js
import { createContainer } from 'react-tracked';
import { call, put, takeLatest } from 'redux-saga/effects';
import useSagaReducer from 'use-saga-reducer';

const initialState = { loading: false, data: null };
const reducer = (state, action) => {
  if (action.type === 'FETCH_STARTED') {
    return { ...state, loading: true };
  } else if (action.type === 'FETCH_FINISHED') {
    return { ...state, loading: false, data: action.data };
  } else {
    return state;
  }
};

function* fetcher(action) {
  yield put({ type: 'FETCH_STARTED' });
  const response = yield call(() => fetch(`https://reqres.in/api/users/${action.id}?delay=1`));
  const data = yield call(() => response.json());
  yield put({ type: 'FETCH_FINISHED', data });
};

function* fetchingSaga() {
  yield takeLatest('FETCH_DATA', fetcher);
}

const useValue = () => useSagaReducer(fetchingSaga, reducer, initialState);
const { Provider, useTracked } = createContainer(useValue);
```

Invoking it is simple.

```js
dispatch({ type: 'FETCH_DATA', id: 1 });
```

Notice the similarity and the difference.
If you are not familiar with generator functions, it may seem weird.

Anyway, try the working example.

<https://codesandbox.io/s/fancy-silence-1pukj>

(Unfortunately, this sandbox doesn't work online as of writing. Please "Export to ZIP" and run locally.)

### useReducerAsync

[use-reducer-async](https://github.com/dai-shi/use-reducer-async) provides a custom hook `useReducerAsync`.
This is the library I developed, inspired by `useSageReducer`.
It's not capable of what generator functions can do,
but it works with any async functions.

The following is the same example with this hook.

```js
import { createContainer } from 'react-tracked';
import { useReducerAsync } from 'use-reducer-async';

const initialState = { loading: false, data: null };
const reducer = (state, action) => {
  if (action.type === 'FETCH_STARTED') {
    return { ...state, loading: true };
  } else if (action.type === 'FETCH_FINISHED') {
    return { ...state, loading: false, data: action.data };
  } else {
    return state;
  }
};

const asyncActionHandlers = {
  FETCH_DATA: (dispatch, getState) => async (action) => {
    dispatch({ type: 'FETCH_STARTED' });
    const response = await fetch(`https://reqres.in/api/users/${action.id}?delay=1`);
    const data = await response.json();
    dispatch({ type: 'FETCH_FINISHED', data });
  },
};

const useValue = () => useReducerAsync(reducer, initialState, asyncActionHandlers);
const { Provider, useTracked } = createContainer(useValue);
```

You can invoke it in the same way.

```js
dispatch({ type: 'FETCH_DATA', id: 1 });
```

The pattern is similar to useSagaReducer, but
the syntax is similar to useThunkReducer or the native solution.

Try the working example.

<https://codesandbox.io/s/bitter-frost-4lxck>

### Comparison

Although it can be biased, here's what I suggest.
If you would like a solution without libraries, use the native one.
If you are saga users, use useSagaReducer with no doubt.
If you like redux-thunk, useThunkReducer would be good.
Otherwise, consider useReducerAsync or the native solution.

For TypeScript users, my recommendations are useSageReducer and useReducerAsync.
The native solution should also work.
Please check out the fully typed examples in React Tracked.

- <https://github.com/dai-shi/react-tracked/tree/master/examples/12_async>
- <https://github.com/dai-shi/react-tracked/tree/master/examples/13_saga>

### Closing notes

To be honest, I think the native solution works fine for small apps.
So, I wasn't so motivated to create a library. However, during
writing a tutorial for React Tracked, I noticed that having a pattern
restricted by a library is easier to explain. use-reducer-async
is a tiny library and it's nothing fancy. But, it shows a pattern.

The other note about async actions is Suspense for Data Fetching.
It's currently in the experimental channel.
The new recommended way of data fetching is Render-as-You-Fetch pattern.
That's totally different from the patterns described in this post.
We will see how it goes. Most likely, that new pattern
requires a library that would ease developers to follow the pattern.
If you are interested, please check out my
[experimental project](https://github.com/dai-shi/react-hooks-fetch).
