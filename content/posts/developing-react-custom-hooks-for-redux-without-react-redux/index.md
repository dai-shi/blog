---
title: "Developing React custom hooks for Redux without react-redux"
subtitle: "An alternative library to react-redux or a feature proposal to react-redux"
date: 2018-12-14T12:00:00+09:00
tags: ["react", "hooks", "typescript", "global-state", "redux", "proxy"]
---

### Background

Ever since I learned Redux, I've been thinking an alternative integration of react and redux to the official react-redux library. The intuition behind it is that Redux is super simple which I like a lot, whereas react-redux is complex because of performance tuning. If you are developing a business product, the official react-redux is to go without a doubt, but if you are playing with creating a toy app or you are just starting learning Redux, you might want something more straightforward.

React will introduce hooks soon, which allow creating stateful logic for function components. They are called custom hooks. There has been many discussions how to use hooks to create new React bindings for Redux. There are also many proposals that would replace Redux at all, including [mine](https://blog.axlight.com/posts/an-alternative-to-react-redux-by-react-hooks-api-for-both-javascript-and-typescript/).

This article describes how to develop custom hooks for Redux (not replacing it) in a straightforward way. We first describe a naive implementation, and later introduce a Proxy approach to seamlessly improve performance to some extent.

### A naive implementation

Let's start with a naive implementation. If you are not familiar with Context API and Hooks API, please visit the official docs to learn them first. The rest of this article assumes the basic understanding of them.

https://reactjs.org/docs/context.html

https://reactjs.org/docs/hooks-intro.html

We create a single context for passing a Redux store. There would be other ways like passing a Redux state or a part of the state for possible further improvements.

```jsx
const ReduxStoreContext = createContext();
const ReduxProvider = ({ store, children }) => (
  <ReduxStoreContext.Provider value={store}>
    {children}
  </ReduxStoreContext.Provider>
);
```

We then define two hooks: `useReduxDispatch` and `useReduxState`. The reason we have separate hooks is that not all components use both hooks at the same time, and we want to hide the implementation how we use context as we might change the implementation in the future.

The implementation of `useReduxDispatch` is very easy.

```javascript
const useReduxDispatch = () => {
  const store = useContext(reduxStoreContext);
  return store.dispatch;
};
```

The naive implementation of `useReduxState` is the following, using four hooks: `useContext`, `useRef`, `useEffect` and `useForceUpdate`.

```javascript
const useReduxState = () => { 
  const forceUpdate = useForceUpdate();
  const store = useContext(ReduxStoreContext);
  const state = useRef(store.getState());
  useEffect(() => {
    const callback = () => {
      state.current = store.getState();
      forceUpdate();
    };
    const unsubscribe = store.subscribe(callback);
    return unsubscribe;
  }, []);
  return state.current;
};
```

Basically, we just subscribe to any changes in the state. (A minor note: this doesn't support changing the store on the fly which may be important for testing.)

Here, `useForceUpdate` is implemented as below [[1](https://github.com/facebook/react/issues/14110#issuecomment-446845886)].

```javascript
const forcedReducer = state => !state;
const useForceUpdate = () => useReducer(forcedReducer, false)[1];
```

### The example code how to use it

Let's see how we use these hooks. It's a bit long but bare with us.

```jsx
const initialState = { count: 0, text: 'hello' };
const reducer = (state = initialState, action) => {
  switch (action.type) {
    case 'inc': return { ...state, count: state.count + 1 };
    case 'setText': return { ...state, text: action.text };
    default: return state;
  }
};

const store = createStore(reducer);

const Counter = () => {
  const state = useReduxState();
  const dispatch = useReduxDispatch();
  const inc = useCallback(() => dispatch({ type: 'inc' }), []);
  return (
    <div>
      <div>Count: {state.count}</div>
      <button onClick={inc}>+1</button>
    </div>
  );
};

const TextBox = () => {
  const state = useReduxState();
  const dispatch = useReduxDispatch();
  const setText = useCallback(event => dispatch({
    type: 'setText',
    text: event.target.value,
  }), []);
  return (
    <div>
      <div>Text: {state.text}</div>
      <input value={state.text} onChange={setText} />
    </div>
  );
};

const App = () => (
  <ReduxProvider store={store}>
    <Counter />
    <TextBox />
  </ReduxProvider>
);
```

If you are familiar with Redux and hooks, there should be little problem reading the code, hopefully. (Again, we assume the store will never be changed, which can be wrong.)

### Improving the custom hook with Proxy

If you are familiar with Redux and react-redux, you notice that this is not ideal. If we click the button in Counter, not only it re-renders the Counter component, but also it re-renders the TextBox component. Well, this is not a problem until it is a problem. For small apps, it just works. For larger apps, it may have a performance problem. Anyway, we want to avoid unnecessary rendering if possible. The `mapStateToProps` function in react-redux is a well known approach for this. Developers specify which part of the state is used for a component. Defining such functions is not only extra work but also troublesome in some cases. It's easy to add heavy computations in such functions unless developers are familiar with memoization. We found beginner developers very hard on it.

What if we don't need to specify anything, meaning no `mapStateToProps`? This is where Proxy comes in. With Proxy, the code can automatically detect which part of the state is used in rendering. Later, if the state is changed and it's notified by the subscription, it can know if the part of the interest is changed and re-render only if necessary.

Let's modify our hook `useReduxState` to implement this feature. For now, we only care the first level of the state object (not deep in the object tree).

```javascript
const useReduxState = () => { 
  const forceUpdate = useForceUpdate();
  const store = useContext(ReduxStoreContext);
  const state = useRef(store.getState());
  const used = useRef({});
  const handler = useMemo(() => ({
    get: (target, name) => {
      used.current[name] = true;
      return target[name];
    },
  }), []);
  useEffect(() => {
    const callback = () => {
      const nextState = store.getState();
      const changed = Object.keys(used.current).find(key => state.current[key] !== nextState[key]);
      if (changed) {
        state.current = nextState;
        forceUpdate();
      }
    };
    const unsubscribe = store.subscribe(callback);
    return unsubscribe;
  }, []);
  return new Proxy(state.current, handler);
};
```

Now, it should not re-render TextBox even if we click the button in Counter in the example, and vice versa.

This implementation is somewhat limited, and you might think that the shallow comparison is not enough. Maybe so, if you have already a concrete design of state structure. However, if you start designing a new state structure, you could design it so that the first-level separation is enough for performance. Also, note that for complex structure where you would need `reselect` in react-redux, you could use `useMemo` in a function component so that it returns a memoized element (ReactNode).

### The library

We developed a library with this approach. This library actually allows deep comparison thanks to [proxyequal](https://www.npmjs.com/package/proxyequal). (Still, the shallow comparison described in the previous section may work better in a certain use case.)

https://github.com/dai-shi/react-hooks-easy-redux

This repository contains several examples. You can also try them online. For example:

[react-hooks-easy-redux-example - CodeSandbox](https://codesandbox.io/s/github/dai-shi/react-hooks-easy-redux/tree/master/examples/02_typescript?module=%2Fsrc%2FCounter.tsx&view=preview)

### Final notes

There could be some edge cases where this approach or the library doesn't work well. I would be happy to hear feedbacks either by Twitter or GitHub issues.

The official react-redux has also discussions about the Proxy approach and they might implement it in the future. Would be nice. We look forward to the possibilities of React Hooks.
