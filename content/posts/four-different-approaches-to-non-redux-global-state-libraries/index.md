---
title: "Four different approaches to non-Redux global state libraries"
subtitle: "From the consuming perspective"
date: 2019-07-06T16:19:00+09:00
tags: ["react", "context", "hooks", "redux"]
---

### Introduction

Since React hooks landed, 
there has been many libraries proposed for global state.
Some of them are simple wrappers around context.
Whereas, some of them are full featured state management systems.

Technically, there are several implementations how to store
state and notify changes.
We don't go in detail in this post, but just note two axes.

1. whether context based or external store
2. whether subscriptions based or context propagation

In this post, we focus on API design of hooks at the consumer end.
In my observation,
there are four approaches to the API design.
Let's see each approach by example in pseudo code.
As a simple example, we assume an app that has the followings.

- two global counters,
- two counter components, and
- an action to increment both counters.

Note that it is implementation agnostic at the provider end.
So, `<Provider>` doesn't necessarily imply React context.

### Approach 1: Multiple contexts

```jsx
const App = () => (
  <Counter1Provider initialState={0}>
    <Counter2Provider initialState={0}>
      <Counter1 />
      <Counter2 />
    </Counter2Provider>
  </Counter1Provider>
);

const Counter1 = () => {
  const [count1, dispatch1] = useCounter1();
  const [, dispatch2] = useCounter2();
  const incrementBoth = () => {
    dispatch1({ type: 'increment' });
    dispatch2({ type: 'increment' });
  };
  return (
    <div>
      <div>Count1: {count1}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};

const Counter2 = () => {
  const [, dispatch1] = useCounter1();
  const [count2, dispatch2] = useCounter2();
  const incrementBoth = () => {
    dispatch1({ type: 'increment' });
    dispatch2({ type: 'increment' });
  };
  return (
    <div>
      <div>Count2: {count2}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};
```

This approach is probably the most idiomatic.
One could easily implement this approach with React context and useContext.

The libraries with this approach:
[constate](https://github.com/diegohaz/constate) and
[unstated-next](https://github.com/jamiebuilds/unstated-next)

### Approach 2: Select by property names (or paths)

```jsx
const App = () => (
  <Provider initialState={{ count1: 0, count2: 0 }}>
    <Counter1 />
    <Counter2 />
  </Provider>
);

const Counter1 = () => {
  const count1 = useGlobalState('count1');
  const dispatch = useDispatch();
  const incrementBoth = () => {
    dispatch({ type: 'incrementBoth' });
  };
  return (
    <div>
      <div>Count1: {count1}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};

const Counter2 = () => {
  const count2 = useGlobalState('count2');
  const dispatch = useDispatch();
  const incrementBoth = () => {
    dispatch({ type: 'incrementBoth' });
  };
  return (
    <div>
      <div>Count2: {count2}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};
```

This approach is to put more values in a single store.
A single store allows to dispatch one action to change multiple values.
You specify a property name to get a corresponding value.
It's simple to specify by a name, but somewhat limited in a complex case.

The libraries with this approach:
[react-hooks-global-state](https://github.com/dai-shi/react-hooks-global-state) and
[shareon](https://github.com/storeon/storeon)

### Approach 3: Select by selector functions

```jsx
const App = () => (
  <Provider initialState={{ count1: 0, count2: 0 }}>
    <Counter1 />
    <Counter2 />
  </Provider>
);

const Counter1 = () => {
  const count1 = useSelector(state => state.count1); // changed
  const dispatch = useDispatch();
  const incrementBoth = () => {
    dispatch({ type: 'incrementBoth' });
  };
  return (
    <div>
      <div>Count1: {count1}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};

const Counter2 = () => {
  const count2 = useSelector(state => state.count2); // changed
  const dispatch = useDispatch();
  const incrementBoth = () => {
    dispatch({ type: 'incrementBoth' });
  };
  return (
    <div>
      <div>Count2: {count2}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};
```

Only two lines are changed from the previous code.
Selector functions are more flexible than property names.
So flexible that it may be misused like doing expensive computations.
Most importantly, performance optimization often requires
to keep object referential equality.

The libraries with this approach:
[zustand](https://github.com/react-spring/zustand) and
[react-sweet-state](https://github.com/atlassian/react-sweet-state)

### Approach 4: State usage tracking

```jsx
const App = () => (
  <Provider initialState={{ count1: 0, count2: 0 }}>
    <Counter1 />
    <Counter2 />
  </Provider>
);

const Counter1 = () => {
  const state = useTrackedState();
  const dispatch = useDispatch();
  const incrementBoth = () => {
    dispatch({ type: 'incrementBoth' });
  };
  return (
    <div>
      <div>Count1: {state.count1}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};

const Counter2 = () => {
  const state = useTrackedState();
  const dispatch = useDispatch();
  const incrementBoth = () => {
    dispatch({ type: 'incrementBoth' });
  };
  return (
    <div>
      <div>Count2: {state.count2}</div>
      <button onClick={incrementBoth}>Increment both</button>
    </div>
  );
};
```

Notice the `state` part is changed from the previous code.
The `dispatch` part is not changed.
This approach eliminates selector functions, and it's hardly misused.
One big concern is performance optimization.
It's out of the scope of this post, but according to some benchmarks,
it's actually fairly good.
Please checkout [the other post](https://blog.axlight.com/posts/benchmark-react-tracked/) if you are interested.

The libraries with this approach:
[react-tracked](https://github.com/dai-shi/react-tracked)

### Closing notes

The example might be too artificial, but I hope it explains the differences.
Personally, I would use any approaches
depending on cases and their requirements.

As a final note, the second purpose of this post is to let readers know
the last approach, "state usage tracking." I hope you get it.
