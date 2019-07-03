---
title: "React hooks useState and useReducer are equivalent in theoretical expressiveness"
subtitle: "Which do you prefer?"
date: 2019-07-01T08:00:00+09:00
tags: ["react", "hooks", "usereducer"]
---

### Introduction

`useReducer` is a powerful hook. It's known that
`useState` is implemented with `useReducer`.

In the React hooks [docs](https://reactjs.org/docs/hooks-reference.html#usereducer), it's noted like this:

> useReducer is usually preferable to useState when you have complex state logic that involves multiple sub-values or when the next state depends on the previous one. useReducer also lets you optimize performance for components that trigger deep updates because you can pass dispatch down instead of callbacks.

For a long time, I misunderstood that useReducer is more powerful than useState
and there's some optimization that can't be achieved by useState.

It turns out that useState is as powerful as useReducder in terms of expressiveness. This is because useState allows [functional updates](https://reactjs.org/docs/hooks-reference.html#functional-updates).
Even with deep updates, you can pass down custom callbacks.

So, whether you useState or useReducer is just your preference.
I would useReducer when I used `dispatch` in JSX. Having logic outside of
JSX seems clean to me.

### Example

If you create a custom hook, whether you useState or useReducer is
just about an internal implementation issue.
Let's look at an example. We implement a simple counter example
with two hooks. For both cases, hooks return action callbacks,
which is important to hide implementation details in this comparison.

##### useReducer

```javascript
const initialState = { count1: 0, count2: 0 };

const reducer = (state, action) => {
  switch (action.type) {
    case 'setCount1':
      if (state.count1 === action.value) return state; // to bail out
      return { ...state, count1: action.value };
    case 'setCount2':
      if (state.count2 === action.value) return state; // to bail out
      return { ...state, count2: action.value };
    default:
      throw new Error('unknown action type');
  }
};

const useCounter = () => {
  const [state, dispatch] = useReducer(reducer, initialState);
  const setCount1 = useCallback(value => {
    dispatch({ type: 'setCount1', value });
  }, []);
  const setCount2 = useCallback(value => {
    dispatch({ type: 'setCount2', value });
  }, []);
  return { ...state, setCount1, setCount2 };
};
```

##### useState

```javascript
const initialState = { count1: 0, count2: 0 };

const useCounter = () => {
  const [state, setState] = useState(initialState);
  const setCount1 = useCallback(value => {
    setState(prevState => {
      if (prevState.count1 === value) return prevState; // to bail out
      return { ...state, count1: value };
    });
  }, []);
  const setCount2 = useCallback(value => {
    setState(prevState => {
      if (prevState.count2 === value) return prevState; // to bail out
      return { ...state, count2: value };
    });
  }, []);
  return { ...state, setCount1, setCount2 };
};
```

Which do you feel comfortable with?

### Bonus

If useState is as powerful as useReducer, useReducer
should be able to be implemented with useState in userland.

```javascript
const useReducer = (reducer, initialArg, init) => {
  const [state, setState] = useState(
    init ? () => init(initialArg) : initialArg,
  );
  const dispatch = useCallback(
    action => setState(prev => reducer(prev, action)),
    [reducer],
  );
  return useMemo(() => [state, dispatch], [state, dispatch]);
};
```

### Final notes

Most of my libraries are written with useReducer,
but I might change my mind a bit, and consider using
useState when it's more appropriate.

I did change my experimental library. The diff is [here](https://github.com/dai-shi/react-hooks-fetch/commit/b20ff29acab2ad2040a5bd6b547d39a43366a868).

<https://github.com/dai-shi/react-hooks-fetch>

One last note, as for unit testing, I'm sure separated `reducer` is easier to test.
