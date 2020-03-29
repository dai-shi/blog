---
title: "Redux in Worker: Off-main-thread Redux Reducers and Middleware"
subtitle: "Examples with async middleware"
date: 2020-03-29T20:00:00+09:00
tags: ["react", "redux", "global-state", "worker"]
---

### Introduction

Redux is a framework-agnostic library for global state.
It's often used with React.

While I like the abstraction of Redux,
React will introduce Concurrent Mode in the near future.
If we want to get benefit of `useTransition`,
state must be inside React to allow state branching.
That means we can't get the benefit with Redux.

I've been developing [React Tracked](https://react-tracked.js.org)
for global state that allows state branching.
It works well in Concurrent Mode.
That leaves me a question: What is a use case that only Redux can do.

The reason Redux can't allow state branching is that the state is
in the external store. So, what is the benefit of having an external store.
[Redux Toolkit](https://redux-toolkit.js.org) can be one answer.
I have another answer, an external store allow off main thread.

React is a UI library, and it's intended to run in the main UI thread.
Redux is usually UI agnostic, so we can run it in a worker thread.

There has been several experiments to off load Redux from the main thread,
and run some or all of Redux work in Web Workers.
I've developed a library for off load the *entire* Redux store.

### redux-in-worker

The library is called redux-in-worker. Please check out the GitHub repository.

<https://github.com/dai-shi/redux-in-worker>

Although this library is not dependent on React,
it's developed with the mind to be used with React.
That is, it will make sure to keep object referential equality,
which allows to prevent unnecessary re-renders in React.

Please check out the blog post I wrote about it.

[Off-main-thread React Redux with Performance](https://blog.axlight.com/posts/off-main-thread-react-redux-with-performance/)

In the next sections, I will show some code to work with async actions
with redux-in-worker.

### redux-api-middleware

[redux-api-middleware](https://github.com/agraboso/redux-api-middleware)
is one of the libraries that existed from the early days.
It receives actions and run API calls described in the actions.
The action object is serializable, so we can send it to the worker
without any problems.

Here's the example code:

```ts
import { createStore, applyMiddleware } from 'redux';
import { apiMiddleware } from 'redux-api-middleware';

import { exposeStore } from 'redux-in-worker';

export const initialState = {
  count: 0,
  person: {
    name: '',
    loading: false,
  },
};

export type State = typeof initialState;

export type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setName'; name: string }
  | { type: 'REQUEST' }
  | { type: 'SUCCESS'; payload: { name: string } }
  | { type: 'FAILURE' };

const reducer = (state = initialState, action: Action) => {
  console.log({ state, action });
  switch (action.type) {
    case 'increment': return {
      ...state,
      count: state.count + 1,
    };
    case 'decrement': return {
      ...state,
      count: state.count - 1,
    };
    case 'setName': return {
      ...state,
      person: {
        ...state.person,
        name: action.name,
      },
    };
    case 'REQUEST': return {
      ...state,
      person: {
        ...state.person,
        loading: true,
      },
    };
    case 'SUCCESS': return {
      ...state,
      person: {
        ...state.person,
        name: action.payload.name,
        loading: false,
      },
    };
    case 'FAILURE': return {
      ...state,
      person: {
        ...state.person,
        name: 'ERROR',
        loading: false,
      },
    };
    default: return state;
  }
};

const store = createStore(reducer, applyMiddleware(apiMiddleware));

exposeStore(store);
```

The above code run in a worker.

The code run in the main thread is the following:

```ts
import { wrapStore } from 'redux-in-worker';
import { initialState } from './store.worker';

const store = wrapStore(
  new Worker('./store.worker', { type: 'module' }),
  initialState,
  window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__(),
);
```

Please find the full example in the repository:

<https://github.com/dai-shi/redux-in-worker/tree/master/examples/04_api>

### redux-saga

Another library that can be used with redux-in-worker is
[redux-saga](https://redux-saga.js.org).
It's a powerful library for any async functions with generators.
Because its action object is serializable, it just works.

Here's the example code:

```ts
import { createStore, applyMiddleware } from 'redux';
import createSagaMiddleware from 'redux-saga';
import {
  call,
  put,
  delay,
  takeLatest,
  takeEvery,
  all,
} from 'redux-saga/effects';

import { exposeStore } from 'redux-in-worker';

const sagaMiddleware = createSagaMiddleware();

export const initialState = {
  count: 0,
  person: {
    name: '',
    loading: false,
  },
};

export type State = typeof initialState;

type ReducerAction =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'SET_NAME'; name: string }
  | { type: 'START_FETCH_USER' }
  | { type: 'SUCCESS_FETCH_USER'; name: string }
  | { type: 'ERROR_FETCH_USER' };

type AsyncActionFetch = { type: 'FETCH_USER'; id: number }
type AsyncActionDecrement = { type: 'DELAYED_DECREMENT' };
type AsyncAction = AsyncActionFetch | AsyncActionDecrement;

export type Action = ReducerAction | AsyncAction;

function* userFetcher(action: AsyncActionFetch) {
  try {
    yield put<ReducerAction>({ type: 'START_FETCH_USER' });
    const response = yield call(() => fetch(`https://jsonplaceholder.typicode.com/users/${action.id}`));
    const data = yield call(() => response.json());
    yield delay(500);
    const { name } = data;
    if (typeof name !== 'string') throw new Error();
    yield put<ReducerAction>({ type: 'SUCCESS_FETCH_USER', name });
  } catch (e) {
    yield put<ReducerAction>({ type: 'ERROR_FETCH_USER' });
  }
}

function* delayedDecrementer() {
  yield delay(500);
  yield put<ReducerAction>({ type: 'DECREMENT' });
}

function* userFetchingSaga() {
  yield takeLatest<AsyncActionFetch>('FETCH_USER', userFetcher);
}

function* delayedDecrementingSaga() {
  yield takeEvery<AsyncActionDecrement>('DELAYED_DECREMENT', delayedDecrementer);
}

function* rootSaga() {
  yield all([
    userFetchingSaga(),
    delayedDecrementingSaga(),
  ]);
}

const reducer = (state = initialState, action: ReducerAction) => {
  console.log({ state, action });
  switch (action.type) {
    case 'INCREMENT': return {
      ...state,
      count: state.count + 1,
    };
    case 'DECREMENT': return {
      ...state,
      count: state.count - 1,
    };
    case 'SET_NAME': return {
      ...state,
      person: {
        ...state.person,
        name: action.name,
      },
    };
    case 'START_FETCH_USER': return {
      ...state,
      person: {
        ...state.person,
        loading: true,
      },
    };
    case 'SUCCESS_FETCH_USER': return {
      ...state,
      person: {
        ...state.person,
        name: action.name,
        loading: false,
      },
    };
    case 'ERROR_FETCH_USER': return {
      ...state,
      person: {
        ...state.person,
        name: 'ERROR',
        loading: false,
      },
    };
    default: return state;
  }
};

const store = createStore(reducer, applyMiddleware(sagaMiddleware));
sagaMiddleware.run(rootSaga);

exposeStore(store);
```

The above code run in a worker.

The code run in the main thread is the following:

```ts
import { wrapStore } from 'redux-in-worker';
import { initialState } from './store.worker';

const store = wrapStore(
  new Worker('./store.worker', { type: 'module' }),
  initialState,
  window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__(),
);
```

This is exactly the same as the previous example.

Please find the full example in the repository:

<https://github.com/dai-shi/redux-in-worker/tree/master/examples/05_saga>

### Closing notes

One of the biggest hurdles in this approach is
[redux-thunk](https://github.com/reduxjs/redux-thunk).
redux-thunk takes a function action which is not serializable.
It's the official tool and included in Redux Toolkit too.
This implies this approach is not going to be mainstream.

But anyway,
I wish somebody likes this approach and evaluates in some real environments.
Please feel free to open a discussion in [GitHub issues](https://github.com/dai-shi/redux-in-worker/issues).

By the way, I have developed another library for React to use Web Workers.

<https://github.com/dai-shi/react-hooks-worker>

This library lets you off-main-thread any functions.
It's a small library and fairly stable. Check it out too.
