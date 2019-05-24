---
title: "React Hooks Tutorial on Developing a Custom Hook for Data Fetching"
subtitle: "Hooks are coming in React 16.7."
date: 2018-11-29T12:00:00+09:00
tags: ["react", "hooks", "async", "fetch"]
---

### Introduction

React Hooks are a new feature coming in React 16.7. It allows us to write stateful function components that were impossible before without class components. The official docs are must-read, so check them out if you haven't.

https://reactjs.org/docs/hooks-intro.html

Besides stateful function components, this new feature allow us to build a custom hook to share logic between components. This has been possible with High-order Components (HoCs), and although there's technically no difference what hooks can achieve, hooks simplify it a lot and reduce so called "wrapper hell". This simplicity encourages building custom hooks, and this trend reminds me of npm's early days.

This article explains how to write a custom hook and shows how it can share logic even if it's a few lines of code. We take a simple data fetching (Fetch API) as an example.

### The goal
Given the readers already learned the basis of React Hooks, we'd start with the goal example how a custom hook we develop would be used.

```jsx
const MyComponent = () => {
  const { error, loading, data } = useFetch('http://...');
  if (error) return <Err error={error} />;
  if (loading) return <Loading />;
  return (
    <DataView data={data} />
  );
};
```

Here, `useFetch` is the custom hook. Isn't it intuitive? `MyComponent` is a "stateful" function component whereas `DataView` is can be a stateless function component which should be more reusable.

### Writing a custom hook

Let's see how we write a custom hook to realize the above goal. A custom hook is just a function that uses some other hooks. We define a function first.

```javascript
const useFetch = (url) => {
  // ...
};
```

This hook returns three values, so we define three state values. Note that we don't need to combine them in one state object.

```javascript
const useFetch = (url) => {
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);
  const [data, setData] = useState(null);
  // ...
  return { error, loading, data };
};
```

Now, the main part is defined in a `useEffect` hook. Notice, we pass `url` in the input array.

```javascript
const useFetch = (url) => {
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);
  const [data, setData] = useState(null);
  useEffect(() => {
    (async () => {
      setLoading(true);
      const response = await fetch(url);
      const data = await response.json();
      setData(data);
      setLoading(false);
    })();
  }, [url]);
  return { error, loading, data };
};
```

We haven't implemented error handling. Let's add it.

```javascript
const useFetch = (url) => {
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);
  const [data, setData] = useState(null);
  useEffect(() => {
    (async () => {
      setLoading(true);
      try {
        const response = await fetch(url);
        if (response.ok) {
          const data = await response.json();
          setData(data);
        } else {
          setError(new Error(response.statusText));
        }
      } catch (e) {
        setError(e);
      }
      setLoading(false);
    })();
  }, [url]);
  return { error, loading, data };
};
```

This is the basic custom hook to achieve our goal. It's reusable in several components. You can check out the working example in this codesandbox.

https://codesandbox.io/s/github/dai-shi/react-hooks-fetch/tree/master/examples/01_minimal

### Extending the custom hook

Our hook is still basic, and we might want to support POST method or non-JSON data. There could be various ways to extend this hook. Let's try a simple way to keep Fetch API as much as possible.

```javascript
const defaultOpts = {};
const useFetch = (input, opts = defaultOpts) => {
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);
  const [data, setData] = useState(null);
  const {
    readBody = body => body.json(),
    ...init
  } = opts;
  useEffect(() => {
    (async () => {
      setLoading(true);
      try {
        const response = await fetch(input, init);
        if (response.ok) {
          const body = await readBody(response);
          setData(body);
        } else {
          setError(new Error(response.statusText));
        }
      } catch (e) {
        setError(e);
      }
      setLoading(false);
    })();
  }, [input, opts]);
  return { error, loading, data };
};
```

One of the difficulties of this extended custom hook is that `opts` is passed in the input array of `useEffect`. Unless users carefully understand how it works, it may cause calling `fetch`es infinitely.

The following is one example how to use this custom hook.

```jsx
const PostRemoteData = () => {
  const opts = useMemo(() => ({
    method: 'POST',
    body: JSON.stringify({
      title: 'foo',
      body: 'bar',
      userId: 1,
    }),
    readBody: body => body.text(),
  }), []);
  const { error, loading, data } = useFetch('http://...', opts);
  if (error) return <Err error={error} />;
  if (loading) return <Loading />;
  return (
    <span>Result: {data}</span>
  );
};
```

### The library

Although this is just a tiny custom hook, I made it as a library and publish it in npmjs.com.

https://github.com/dai-shi/react-hooks-fetch

### Final notes

Actually, I'm not quite sure if Fetch API is an appropriate topic to explain building a custom hook. People might prefer data fetching layers isolated from view layers. Nevertheless, I hope this article explains how to develop a custom hook, and helps noticing its reusability.
