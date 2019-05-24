---
title: "Developing React custom hooks for abortable async functions with AbortController"
subtitle: "Seek possibilities of React Hooks."
date: 2018-12-31T12:00:00+09:00
tags: ["react", "hooks", "async", "abortable"]
---

In [my previous article](https://blog.axlight.com/posts/introduction-to-abortable-async-functions-for-react-with-hooks/), I introduced how the custom hook `useAsyncTask` handles async functions with AbortController and demonstrated a typeahead search example. In this article, I explain about the implementation of `useAsyncTask`.

### Recap

JavaScript promise is not abortable. The fetch API is based on promise, and hence you can't cancel it in pure JavaScript. To cancel `fetch`, the DOM spec introduced `AbortController`. The AbortController is a general interface and not specific to `fetch`. Technically, we can use it to cancel promises, and it would be nice to have an easy way to handle abortable async functions. In the React world, we are expecting the Hooks API soon. I've started a project to implement custom hooks for abortable async functions.

https://github.com/dai-shi/react-hooks-async

https://www.npmjs.com/package/react-hooks-async

### The implementation of useAsyncTask

The `useAsyncTask` hook is to create an async task that can be started by `useAsyncRun` which we describe later in this article. To create an async task, you need to pass a function that receives an AbortController instance and returns an abortable promise. As a concrete example, we describe how to implement a custom hook for fetch later.

The main part of the code is the following.

```javascript
const useAsyncTask = (func, inputs) => {
  const forceUpdate = useForceUpdate();
  const task = useRef({});
  const newTask = useMemo(() => createTask(func, (t) => {
    if (task.current && task.current.taskId === t.taskId) {
      task.current = t;
      forceUpdate();
    }
  }), inputs);
  if (task.current && task.current.taskId !== newTask.taskId) {
    task.current = newTask;
  }
  useEffect(() => {
    const cleanup = () => { task.current = null; };
    return cleanup;
  }, []);
  return task.current;
};
```

Three notes to read this code:

- the `task` object is defined by `useRef`, and re-rendering is controlled by `forceUpdate`.
- `createTask` is the core part which takes a callback function in the second argument, and use `useMemo` to call it only `inputs` are changed.
- `useEffect` is to clean up the `task` object in order not to try rendering after unmounted.

The `useForceUpdate` hook is a general hook by the following impelementation.

```javascript
const forcedReducer = state => !state;
const useForceUpdate = () => useReducer(forcedReducer, false)[1];
```

The code of `createTask` is a bit long.

```javascript
const createTask = (func, notifyUpdate) => {
  const taskId = Symbol(`async_task_id_${idCounter += 1}`);
  let abortController = null;
  let task = {
    taskId,
    started: false,
    pending: true,
    error: null,
    result: null,
    start: async () => {
      if (task.started) return;
      abortController = new AbortController();
      task = { ...task, started: true };
      notifyUpdate(task);
      try {
        const result = await func(abortController);
        task = { ...task, pending: false, result };
      } catch (e) {
        task = { ...task, pending: false, error: e };
      }
      notifyUpdate(task);
    },
    abort: () => {
      if (abortController) {
        abortController.abort();
      }
    },
  };
  return task;
};
```

- the `task` object has `taskId` and four properties: `started`, `pending`, `error` and `result`.
- in addition to them, the `task` object contains `start()` and `abort()` functions to control the execution.
- when `start()` is called, it calls `notifyChange()` back according to asynchronous state changes.
- the first argument `func` is the function which receives an AbortController instance and returns a promise. The `func` is responsible to handle the AbortController correctly.

### The implementation of useAsyncRun

The `useAsyncTask` hook is just to create an async task and make it ready to be started. The `useAsyncRun` hook is the one to actually start the async task. The reason we split the logic into two hooks is for allowing to combine multiple async tasks. The `useAsyncCombineSeq` hook is the one for combining, but we don't go further in this article.

This code is rather simple.

```javascript
const useAsyncRun = (asyncTask) => {
  useEffect(() => {
    if (asyncTask) {
      asyncTask.start();
    }
    const cleanup = () => {
      if (asyncTask) {
        asyncTask.abort();
      }
    };
    return cleanup;
  }, [asyncTask && asyncTask.taskId]);
};
```

It simply calls `start()` and `abort()`. Notice the input array passed to the second argument of `useEffect`.

### The implementation of useAsyncTaskFetch

The `useAsyncTaskFetch` hook is a wrapper function of `useAsyncTask`. The basic implementation is the following. The full-featured one is [here](https://github.com/dai-shi/react-hooks-async/blob/master/src/use-async-task-fetch.js) in the GitHub repository.

```javascript
const useAsyncTaskFetch = url => useAsyncTask(
  async (abortController) => {
    const response = await fetch(url, {
      signal: abortController.signal,
    });
    if (!response.ok) {
      throw new Error(response.statusText);
    }
    const body = await response.json();
    return body;
  },
  [url],
);
```

Notice the AbortController signal is passed to `fetch`. This is because the Fetch API supports AbortController. For others, you need to implement handling it. For example, please check out how `useAsyncTaskAxios` is implemented [here](https://github.com/dai-shi/react-hooks-async/blob/master/src/use-async-task-axios.js).

### Summary

This article showed how `useAsyncTask` and other hooks are implemented. I hope they are straightforward with the convention of the use of AbortController. I'm looking for other use cases than `fetch` and would be happy for any feedback.
