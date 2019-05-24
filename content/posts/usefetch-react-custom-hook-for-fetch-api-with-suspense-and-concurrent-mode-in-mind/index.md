---
title: "useFetch: React custom hook for Fetch API with Suspense and Concurrent Mode in Mind"
subtitle: "Until we have react-cache or for a different purpose."
date: 2019-02-04T12:00:00+09:00
tags: ["react", "hooks", "async", "fetch", "concurrent"]
---

### Introduction

While I don't feel like coding React without hooks, react-cache still seems to be still far away. Surely, caching in data fetching important, nevertheless I would like to seek possibilities of implementations only with React Hooks. These might be obsoleted in the future, but I want something today, that is "useFetch". This could be still useful without react-cache in case sophisticated caching is not necessary.

I've started developing "useFetch" as a tutorial to build a custom hook.

[React Hooks Tutorial on Developing a Custom Hook for Data Fetching](https://blog.axlight.com/posts/react-hooks-tutorial-on-developing-a-custom-hook-for-data-fetching/)

Since then, the support for AbortController is added, which is incredible if you are fetching one-time-only data like typeahead suggest.

Now, to make this "useFetch" hook more usable, this article describes about supporting Suspense and also Concurrent React a bit.

### React Suspense

[Code-Splitting - React](https://reactjs.org/docs/code-splitting.html#suspense)

Suspense is currently described in the Code-Splitting section. However, its general purpose is to catch a promise in a component tree and render a fallback element. When the promise is resolved, it will render a normal tree. The good part of Suspense is that you can place anywhere up in the tree, and you could even place only one Suspense at the top.

One uncertain point is whether this functionality will be the same when Suspense for data fetching is released. Never mind. We should be able to follow the upcoming changes.

### Concurrent React

[React 16.x Roadmap - React Blog](https://reactjs.org/blog/2018/11/27/react-16-roadmap.html#react-16x-q2-2019-the-one-with-concurrent-mode)

There is no documentation about Concurrent Mode yet. As far as I understand, what we need to care is that a) the render function can be called more than once, and b) the render phase and the commit phase is not synchronous. As for a), the render function should be pure or at least it shouldn't change the world if it’s called twice. For b), we shouldn’t assume the call order of useEffect of multiple components. I may still misunderstand something, though. I hope some guidelines for React hooks to be available in the future.

### The library

https://github.com/dai-shi/react-hooks-fetch

Let's look at how "useFetch" is implemented. If you would like to see how it’s used, jump to the next section.

Firstly, there's some utility functions, and two notable ones:

```javascript
const useForceUpdate = () => useReducer(state => !state, false)[1];
```

This is a custom hook to force re-rendering.

```javascript
const createPromiseResolver = () => {
  let resolve;
  const promise = new Promise((r) => { resolve = r; });
  return { resolve, promise };
};
```

This is a naive way to create promise that can be resolved from outside.

Next is the main part of the hook:

```javascript
export const useFetch = (input, opts = defaultOpts) => {
  const forceUpdate = useForceUpdate();
  const error = useRef(null);
  const loading = useRef(false);
  const data = useRef(null);
  const promiseResolver = useMemo(createPromiseResolver, [input, opts]);
```

We define some variables with `useRef`, and promiseResolver with `useMemo`.

```javascript
  // ...continued
  useEffect(() => {
    let finished = false;
    const abortController = new AbortController();
    (async () => {
      if (!input) return;
      // start fetching
      loading.current = true;
      forceUpdate();
      const onFinish = (e, d) => {
        if (!finished) {
          finished = true;
          error.current = e;
          data.current = d;
          loading.current = false;
        }
      };
      try {
        const { readBody = defaultReadBody, ...init } = opts;
        const response = await fetch(input, {
          ...init,
          signal: abortController.signal,
        });
        // check response
        if (response.ok) {
          const body = await readBody(response);
          onFinish(null, body);
        } else {
          onFinish(createFetchError(response), null);
        }
      } catch (e) {
        onFinish(e, null);
      }
      // finish fetching
      promiseResolver.resolve();
    })();
    const cleanup = () => {
      if (!finished) {
        finished = true;
        abortController.abort();
      }
      error.current = null;
      loading.current = false;
      data.current = null;
    };
    return cleanup;
  }, [input, opts]);
```

Hmm, too long. Some notes: a) `forceUpdate()` is called when variables are updated. b) At the end of async function, `primiseResolver.resolve()` is called. c) the `cleanup` is to abort running fetch and clear variables. d) the input array of `useEffect` is `[input, opts]`.

```javascript
  // ...continued
  if (loading.current) throw promiseResolver.promise;
  return {
    error: error.current,
    data: data.current,
  };
};
```

Finally, it returns some of the variables, but before that if it's still in the loading state, it throws a promise so that Suspense catches it and show a fallback loading component.

### How to use it

Let me show the minimal example.

```jsx
import React, { Suspense } from 'react';
import ReactDOM from 'react-dom';

import { useFetch } from 'react-hooks-fetch';

const Err = ({ error }) => <span>Error:{error.message}</span>;

const DisplayRemoteData = () => {
  const url = 'https://jsonplaceholder.typicode.com/posts/1';
  const { error, data } = useFetch(url);
  if (error) return <Err error={error} />;
  if (!data) return null;
  return <span>RemoteData:{data.title}</span>;
};

const App = () => (
  <Suspense fallback={<span>Loading...</span>}>
    <DisplayRemoteData />
  </Suspense>
);

ReactDOM.render(<App />, document.getElementById('app'));
```

The component that uses `useFetch` does not handle the loading state and `Suspense` outside of the component does it and shows loading state. One important note is the line `if (!data) return null;` which is required because it's null at first. This is not very nice at the moment, and we would like to find out if there’s a workaround.

You can try this example in codesandbox.

[react-hooks-fetch-example - CodeSandbox](https://codesandbox.io/s/github/dai-shi/react-hooks-fetch/tree/master/examples/01_minimal)

There are some other examples you can find in the GitHub repo with the link to codesandbox.

### Final notes

At first, I was hoping to use both Suspense and ErrorBoundary to show the loading state and the error state so that the main component with `useFetch` only cares about successful case. It turns out that it involves more or less hacks, and I ended up with normal error variable return by a hook. There's also a note in the [doc](https://reactjs.org/docs/react-component.html#error-boundaries):

> Only use error boundaries for recovering from unexpected exceptions; **don't try to use them for control flow.**

Suspense is also hard to work with AbortController. At the point of writing it's pretty much like a hack. For those who are interested, the code is [here](https://github.com/dai-shi/react-hooks-fetch/blob/0fbd03e6581a3e8e9ef13b4ce72ae3c2101ebc5a/examples/04_abort/src/App.tsx). I really want to find a better way for this. I’m curious how react-cache handle aborting. Until then, we continue to improve it.

### Changelog

- _[2019-02-04]:_ Initial publication.
- _[2019-02-05]:_ Improved code simplicity.
