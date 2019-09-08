---
title: "How I Developed React Hooks for Web Workers"
subtitle: "Making Use of Async Generators"
date: 2019-09-08T16:00:00+09:00
tags: ["react", "hooks", "worker"]
---

### Introduction

I have been developing several react hooks libraries.
They provide custom hooks for certain purposes.
One of them is for web workers.
I started it for fun. I got some feedbacks and improved.
This post shows the current implementation which aims the use in production.

In this field, [Comlink](https://github.com/GoogleChromeLabs/comlink)
provides a nice transparent API with Proxies.
Some might have already tried it with React.
I have two reasons why I don't use it for my library.

1. React hooks is reactive by nature, so no async interface is required.
With Comlink, the API in the main thread is an async function.
You need to put `await` in front of `Comlink.wrap`.
With React, we can hide the async behavior in hooks.

2. The RPC style is limited.
Web Workers are often used for time consuming tasks.
We may need to show progress of the tasks or intermediate results for better UX.

### Library

I developed a library to provide a custom hook to use workers easily.
It has zero dependencies and the code is tiny.

<https://github.com/dai-shi/react-hooks-worker>

### Basic Usage

Here's a basic example for calculating fibonacci numbers.
You need two files for worker thread and main thread.
The library exports two functions for each files.

The worker file looks like this.

```javascript
// fib.worker.js

import { exposeWorker } from 'react-hooks-worker';

const fib = i => (i <= 1 ? i : fib(i - 1) + fib(i - 2));

exposeWorker(fib);
```

The react file looks like this.

```jsx
// App.jsx

import React from 'react';
import { useWorker } from 'react-hooks-worker';

const createWorker = () => new Worker('./fib.worker', { type: 'module' });

const CalcFib = ({ count }) => {
  const { result, error } = useWorker(createWorker, count);
  if (error) return <div>Error: {error}</div>;
  return <div>Result: {result}</div>;
};

export const App = () => (
  <div>
    <CalcFib count={5} />
  </div>
);
```

### Async Generators

As I implied, this library provides non-RPC interface.
We use (async) generators to return intermediate states.

Here's an example to show calculation steps of fibonacci numbers.

```javascript
// fib-steps.worker.js

import { exposeWorker } from 'react-hooks-worker';

async function* fib(x) {
  let x1 = 0;
  let x2 = 1;
  let i = 0;
  while (i < x) {
    yield `(calculating...) ${x1}`;
    await new Promise(r => setTimeout(r, 100));
    [x1, x2] = [x2, x1 + x2];
    i += 1;
  }
  yield x1;
}

exposeWorker(fib);
```

### The implementation

The implementation of `exposeWorker` is surprisingly simple.

```javascript
export const exposeWorker = (func) => {
  self.onmessage = async (e) => {
    const r = func(e.data);
    if (r[Symbol.asyncIterator]) {
      for await (const i of r) self.postMessage(i);
    } else if (r[Symbol.iterator]) {
      for (const i of r) self.postMessage(i);
    } else {
      self.postMessage(await r);
    }
  };
};
```

The implementation of `useWorker` can be in various styles.
Currently, it's implemented with useReducer.

```javascript
import {
  useEffect,
  useMemo,
  useRef,
  useReducer,
} from 'react';

const initialState = { result: null, error: null };
const reducer = (state, action) => {
  switch (action.type) {
    case 'init':
      return initialState;
    case 'result':
      return { result: action.result, error: null };
    case 'error':
      return { result: null, error: 'error' };
    case 'messageerror':
      return { result: null, error: 'messageerror' };
    default:
      throw new Error('no such action type');
  }
};

export const useWorker = (createWorker, input) => {
  const [state, dispatch] = useReducer(reducer, initialState);
  const worker = useMemo(createWorker, [createWorker]);
  const lastWorker = useRef(null);
  useEffect(() => {
    lastWorker.current = worker;
    let dispatchSafe = action => dispatch(action);
    worker.onmessage = e => dispatchSafe({ type: 'result', result: e.data });
    worker.onerror = () => dispatchSafe({ type: 'error' });
    worker.onmessageerror = () => dispatchSafe({ type: 'messageerror' });
    const cleanup = () => {
      dispatchSafe = () => null; // we should not dispatch after cleanup.
      worker.terminate();
      dispatch({ type: 'init' });
    };
    return cleanup;
  }, [worker]);
  useEffect(() => {
    lastWorker.current.postMessage(input);
  }, [input]);
  return state;
};
```

An important note: If `createWorker` is referentially different from
the previous one, it stops the previous worker and starts a new one.
Otherwise, it re-uses the worker instance.
There's currently no way to distinguish the results by
multiple invocations to a single worker instance.

### Closing notes

If we use workers for non trivial use cases,
we would likely be using some libraries in workers.
This requires a bundler support.
Until now, I have only tried with
[worker-plugin](https://github.com/GoogleChromeLabs/worker-plugin)
in webpack.
There are other plugins in webpack.
Other bundlers support the similar feature.
You are welcome to try them out and report the result to the project.
