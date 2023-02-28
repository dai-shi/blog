---
title: "Steps to Develop Global State for React With Hooks Without Context"
subtitle: "Support Concurrent Mode"
date: 2020-02-18T23:00:00+09:00
tags: ["react", "hooks", "typescript", "global-state", "redux"]
---

### Introduction

Developing with React hooks is fun for me.
I have been developing several libraries.
The very first library was a library for global state.
It's naively called "react-hooks-global-state" which
turns out to be too long to read.

The initial version of the library was published in Oct 2018.
Time has passed since then, I learned a lot, and now
v1.0.0 of the library is published.

<https://github.com/dai-shi/react-hooks-global-state>

This post shows simplified versions of the code step by step.
It would help understand what this library is aiming at,
while the real code is a bit complex in TypeScript.

### Step 1: Global variable

```js
let globalState = {
  count: 0,
  text: 'hello',
};
```

Let's have a global variable like the above.
We assume this structure throughout this post.
One would create a React hook to read this global variable.

```js
const useGlobalState = () => {
  return globalState;
};
```

This is not actually a React hook
because it doesn't depend on any React primitive hooks.

Now, this is not what we usually want, because it doesn't re-render
when the global variable changes.

### Step 2: Re-render on updates

We need to use React `useState` hook to make it reactive.

```js
const listeners = new Set();

const useGlobalState = () => {
  const [state, setState] = useState(globalState);
  useEffect(() => {
    const listener = () => {
      setState(globalState);
    };
    listeners.add(listener);
    listener(); // in case it's already changed
    return () => listeners.delete(listener); // cleanup
  }, []);
  return state;
};
```

This allows to update React state from outside.
If you update the global variable, you need to notify listeners.
Let's create a function for updating.

```js
const setGlobalState = (nextGlobalState) => {
  globalState = nextGlobalState;
  listeners.forEach(listener => listener());
};
```

With this, we can change `useGlobalState` to return a tuple like `useState`.

```js
const useGlobalState = () => {
  const [state, setState] = useState(globalState);
  useEffect(() => {
    // ...
  }, []);
  return [state, setGlobalState];
};
```

### Step 3: Container

Usually, the global variable is in a file scope.
Let's put it in a function scope to narrow down the scope a bit
and make it more reusable.

```js
const createContainer = (initialState) => {
  let globalState = initialState;
  const listeners = new Set();

  const setGlobalState = (nextGlobalState) => {
    globalState = nextGlobalState;
    listeners.forEach(listener => listener());
  };

  const useGlobalState = () => {
    const [state, setState] = useState(globalState);
    useEffect(() => {
      const listener = () => {
        setState(globalState);
      };
      listeners.add(listener);
      listener(); // in case it's already changed
      return () => listeners.delete(listener); // cleanup
    }, []);
    return [state, setGlobalState];
  };

  return {
    setGlobalState,
    useGlobalState,
  };
};
```

We don't go in detail about TypeScript in this post,
but this form allows to annotate types of `useGlobalState`
by inferring types of `initialState`.

### Step 4: Scoped access

Although we can create multiple containers,
usually we put several items in a global state.

Typical global state libraries have some functionality
to scope only a part of the state.
For example, React Redux uses selector interface to
get a derived value from a global state.

We take a simpler approach here,
which is to use a string key of a global state.
In our example, it's like `count` and `text`.

```js
const createContainer = (initialState) => {
  let globalState = initialState;
  const listeners = Object.fromEntries(Object.keys(initialState).map(key => [key, new Set()]));

  const setGlobalState = (key, nextValue) => {
    globalState = { ...globalState, [key]: nextValue };
    listeners[key].forEach(listener => listener());
  };

  const useGlobalState = (key) => {
    const [state, setState] = useState(globalState[key]);
    useEffect(() => {
      const listener = () => {
        setState(globalState[key]);
      };
      listeners[key].add(listener);
      listener(); // in case it's already changed
      return () => listeners[key].delete(listener); // cleanup
    }, []);
    return [state, (nextValue) => setGlobalState(key, nextValue)];
  };

  return {
    setGlobalState,
    useGlobalState,
  };
};
```

We omit the use of useCallback in this code for simplicity,
but it's generally recommended for a library.

### Step 5: Functional Updates

React `useState` allows [functional updates](https://reactjs.org/docs/hooks-reference.html#functional-updates).
Let's implement this feature.

```js
  // ...

  const setGlobalState = (key, nextValue) => {
    if (typeof nextValue === 'function') {
      globalState = { ...globalState, [key]: nextValue(globalState[key]) };
    } else {
      globalState = { ...globalState, [key]: nextValue };
    }
    listeners[key].forEach(listener => listener());
  };

  // ...
```


### Step 6: Reducer

Those who are familiar with Redux may prefer reducer interface.
React hook useReducer also has basically the same interface.

```js
const createContainer = (reducer, initialState) => {
  let globalState = initialState;
  const listeners = Object.fromEntries(Object.keys(initialState).map(key => [key, new Set()]));

  const dispatch = (action) => {
    const prevState = globalState;
    globalState = reducer(globalState, action);
    Object.keys((key) => {
      if (prevState[key] !== globalState[key]) {
        listeners[key].forEach(listener => listener());
      }
    });
  };

  // ...

  return {
    useGlobalState,
    dispatch,
  };
};
```

### Step 7: Concurrent Mode

In order to get benefits from Concurrent Mode,
we need to use React state instead of an external variable.
The current solution to it is to link a React state to our global state.

The implementation is very tricky, but in essence we create a hook
to create a state and link it.

```js
  const useGlobalStateProvider = () => {
    const [state, dispatch] = useReducer(patchedReducer, globalState);
    useEffect(() => {
      linkedDispatch = dispatch;
      // ...
    }, []);
    const prevState = useRef(state);
    Object.keys((key) => {
      if (prevState.current[key] !== state[key]) {
        // we need to pass the next value to listener
        listeners[key].forEach(listener => listener(state[key]));
      }
    });
    prevState.current = state;
    useEffect(() => {
      globalState = state;
    }, [state]);
  };
```

The `patchedReducer` is required to allow `setGlobalState`
to update global state.
The `useGlobalStateProvider` hook should be used in a stable component
such as an app root component.

Note that this is not a well-known technique,
and there might be some limitations.
For instance, invoking listeners in render is not actually recommended.

To support Concurrent Mode in a proper way, we would need core support.
Currently, `useMutableSource` hook is proposed in
[this RFC](https://github.com/reactjs/rfcs/pull/147).

### Closing notes

This is mostly how [react-hooks-global-state](https://github.com/dai-shi/react-hooks-global-state)
is implemented.
The real code in the library is a bit more complex in TypeScript,
contains `getGlobalState` for reading global state from outside, and
has limited support for Redux middleware and DevTools.

Finally, I have developed some other libraries around global state
and React context, as listed below.

- <https://github.com/dai-shi/reactive-react-redux>
- <https://github.com/dai-shi/react-tracked>
- <https://github.com/dai-shi/use-context-selector>

