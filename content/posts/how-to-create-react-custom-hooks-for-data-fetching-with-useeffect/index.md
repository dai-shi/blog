---
title: "How to create React custom hooks for data fetching with useEffect"
subtitle: "Introduction to the react-hooks-async library"
date: 2019-04-03T12:00:00+09:00
tags: ["react", "hooks", "async", "fetch", "abortable"]
---

### Background

React 16.8 added a new API, Hooks. If you haven't learned hooks yet, go to the official site and read the entire document before continuing this article.

https://reactjs.org/docs/hooks-intro.html

Hooks are a new addition in React 16.8. They let you use state and other React features without writing a class. This…
reactjs.org
This article is about how to create custom hooks for data fetching. As described in [the roadmap](https://reactjs.org/blog/2018/11/27/react-16-roadmap.html), React is planning to release react-cache and Suspense for data fetching in the near future. This is going to be a standard way of data fetching in React, however, data fetching with useEffect is still useful in certain use cases where the lifecycle of fetched data is the same as that of components. In such use cases, caching is not important and we can safely store fetched data in a component local state.

This article describes about a naive implementation of a custom hook for data fetching, its limitation, an API proposal for abortable async functions and the library implemented for the API.

### A naive custom hook

If you have a good understanding of React hooks, the implementation like below can easily be come up with.

```javascript
const useFetch = (url) => {
  const [data, setData] = useState(null);
  useEffect(() => {
    (async () => {
      const res = await fetch(url);
      const data = await res.json();
      setData(data);
    })();
  }, [url]);
  return data;
};
```

Note that error handling and loading state are omitted in this code for simplicity.

The data with this hook only lives in a component state, and is discarded when the component is unmounted. To avoid setting state for unmounted components, you typically introduce a flag like the following.

```javascript
const useFetch = (url) => {
  const [data, setData] = useState(null);
  useEffect(() => {
    let mounted = true;
    (async () => {
      const res = await fetch(url);
      const data = await res.json();
      if (mounted) setData(data);
    })();
    const cleanup = () => { mounted = false; };
    return cleanup;
  }, [url]);
  return data;
};
```

This does work, but it doesn't actually stop data fetching. It would be better if we can really stop data fetching in the browser. [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) enables it and the code becomes like the following.

```javascript
const useFetch = (url) => {
  const [data, setData] = useState(null);
  useEffect(() => {
    let mounted = true;
    const abortController = new AbortController();
    (async () => {
      const res = await fetch(url, {
        signal: abortController.signal,
      });
      const data = await res.json();
      if (mounted) setData(data);
    })();
    const cleanup = () => {
       mounted = false;
       abortController.abort();
    };
    return cleanup;
  }, [url]);
  return data;
};
```

### A general API for abortable async functions

Let's generalize a bit to make any async functions abortable with AbortController. We define an API that is a function which receives an instance of AbortController in the first argument and returns a promise.

With this API, the fetch is implemented something like this:

```javascript
// this is pseudo code
const abortableFetch = (url) => async (abortController) => {
  const res = await fetch(url, { signal: abortController.signal });
  const data = await res.json();
  return data;
};
```

Similarly, [axios](https://github.com/axios/axios), which has a different cancellation mechanism, can be implemented something like this:

```javascript
// this is pseudo code
const abortableAxios = (url) => async (abortController) => {
  const source = axios.CancelToken.source();
  abortController.signal.addEventListener('abort', () => {
    source.cancel('canceled');
  );
  const { data } = await axios({ url, cancelToken: source.token });
  return data;
};
```

### Custom hooks for abortable async functions

We now create custom hooks with this API. There are two hooks; the one is called `useAsyncTask` to prepare an async function ready to start, and the other is called `useAsyncRun` is to actually start it. The object `task` returned by `useAsyncTask` contains the state of an async function and methods to start it and abort it.

### The useAsyncTask hook

Now, we implement the first hook. In addition to the result of an async function, we handle the pending state and the error.

The initial `task` is defined as follows.

```javascript
const initialTask = {
  started: false, // if this async function is started
  pending: true,  // if this async function is not finished
  error: null,    // error of this async function
  result: null,   // result of this async function
  start: null,    // a method to start this async function
  abort: null,    // a method to abort this async function
};
```

We use the terms `pending` and `result` instead of `loading` and `data` because the `task` is not only for data fetching.

The reducer used with useReducer for the `task` is defined as follows.

```javascript
const reducer = (task, action) => {
  switch (action.type) {
    case 'init':
      return initialTask;
    case 'ready':
      return {
        ...task,
        start: action.start,
        abort: action.abort,
      };
    case 'start':
      return {
        ...task,
        started: true,
      };
    case 'result':
      return {
        ...task,
        pending: false,
        result: action.result,
      };
    case 'error':
      return {
        ...task,
        pending: false,
        error: action.error,
      };
    default:
      throw new Error(`unexpected action type: ${action.type}`);
  }
};
```

With these above, our custom hook is implemented.

```javascript
const useAsyncTask = (func, deps) => {
  const [state, dispatch] = useReducer(reducer, initialTask);
  useEffect(() => {
    let dispatchSafe = action => dispatch(action);
    let abortController = null;
    const start = async () => {
      if (abortController) return;
      abortController = new AbortController();
      dispatchSafe({ type: 'start' });
      try {
        const result = await func(abortController);
        dispatchSafe({ type: 'result', result });
      } catch (e) {
        dispatchSafe({ type: 'error', error: e });
      }
    };
    const abort = () => {
      if (abortController) {
        abortController.abort();
      }
    };
    dispatch({ type: 'ready', start, abort });
    const cleanup = () => {
      dispatchSafe = () => null; // avoid to dispatch after stopped
      dispatch({ type: 'init' });
    };
    return cleanup;
  }, deps);
  return state;
};
```

In this implementation, we simply receive `deps` as a second argument and pass it to useEffect. This is a design choice to avoid the use of [useMemo](https://reactjs.org/docs/hooks-reference.html#usememo), which is not recommended for a semantic guarantee.

### The useAsyncRun hook

The second hook is implemented relatively easily.

```javascript
const useAsyncRun = (asyncTask) => {
  const start = asyncTask && asyncTask.start;
  const abort = asyncTask && asyncTask.abort;
  useEffect(() => {
    if (start) start();
    const cleanup = () => {
      if (abort) abort();
    };
    return cleanup;
  }, [start]);
};
```

This hook is not mandatory to use. You could just call the start and abort methods in the task object in event callbacks. Ref: [example](https://github.com/dai-shi/react-hooks-async/tree/master/examples/03_startbutton).

### The useFetch hook

Based on the two hooks described above, we can implement `useAsyncTaskFetch` and `useFetch`.

```javascript
const defaultInit = {};
const defaultReadBody = body => body.json();

const useAsyncTaskFetch = (
  input,
  init = defaultInit,
  readBody = defaultReadBody,
) => useAsyncTask(
  async (abortController) => {
    const response = await fetch(input, {
      signal: abortController.signal,
      ...init,
    });
    if (!response.ok) {
      throw new Error(response.statusText);
    }
    const body = await readBody(response);
    return body;
  },
  [input, init, readBody],
);

const useFetch = (...args) => {
  const asyncTask = useAsyncTaskFetch(...args);
  useAsyncRun(asyncTask);
  return asyncTask;
};
```

The useAxios hook can also be implemented similarly. Ref: [code](https://github.com/dai-shi/react-hooks-async/blob/master/src/use-async-task-axios.js).

### Demo

<iframe style="width: 100%; height: 400px;" src="https://stackblitz.com/edit/react-2kfuyr?ctl=1&embed=1&file=index.js&view=preview"></iframe>

### The react-hooks-async library

The implementation of hooks described in this article is available as a library.

https://github.com/dai-shi/react-hooks-async

This library is not only for data fetching. It provides a general API for abortable async functions, and some utility functions. One notable use case is typeahead search. Check out the examples in the repo for more information.

### Final thoughts

There are many proposals and implementations for data fetching with useEffect, and React might be going to provide one officially. Implementing one by yourself is possible but not trivial. Selecting one out of various libraries is not trivial either. I hope this article helps understand how to implement abortable fetch with hooks. I look forward to any feedback about the library by Twitter or GitHub issues.
