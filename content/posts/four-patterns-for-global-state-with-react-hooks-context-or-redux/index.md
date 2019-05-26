---
title: "Four patterns for global state with React hooks: Context or Redux"
subtitle: "And the libraries I developed"
date: 2019-05-27T00:50:00+09:00
tags: ["react", "context", "hooks", "global-state", "redux"]
---

### Introduction

Global state or shared state is one of the biggest issues
when you start developing a React app.
Should we use Redux? Do hooks provide a Redux-like solution?
I would like to show four patterns toward using Redux.
This is my personal opinion and mainly for new apps.

### Pattern 1: Prop passing

Some might think it wouldn't scale, but the most basic pattern
should still be prop passing. If the app is small enough,
define local state in a parent component and simply pass it
down to child components. I would tolerate two level passing,
meaning one intermediate component.

```jsx
const Parent = () => {
  const [stateA, dispatchA] = useReducer(reducerA, initialStateA);
  return (
    <>
      <Child1 stateA={stateA} dispatchA={dispatchA} />
      <Child2 stateA={stateA} dispatchA={dispatchA} />
    </>
  );
};

const Child1 = ({ stateA, dispatchA }) => (
  ...
);

const Child2 = ({ stateA, dispatchA }) => (
  <>
    <GrandChild stateA={stateA} dispatchA={dispatchA} />
  </>
);

const GrandChild = ({ stateA, dispatchA }) => (
  ...
);
```

### Pattern 2: Context

If an app needs to share state among components
that are more deep than two level,
it's time to introduce context.
Context itself doesn't provide global state functionality,
but combining local state and passing by context does the job.

```jsx
const ContextA = createContext(null);

const Parent = () => {
  const [stateA, dispatchA] = useReducer(reducerA, initialStateA);
  const valueA = useMemo(() => [stateA, dispatchA], [stateA]);
  return (
    <ContextA.Provider value={valueA}>
      <Child1 />
    </ContextA.Provider>
  );
};

const Child1 = () => (
  <GrandChild1 />
);

const GrandChild1 = () => (
  <GrandGrandChild1 />
);

const GrandGrandChild1 = () => {
  const [stateA, dispatchA] = useContext(ContextA);
  return (
    ...
  );
};
```

Note that all components with `useContext(ContextA)` will re-render
if `stateA` is changed, even if it's only a tiny part of the state.
Hence, it's not recommended to use a context for multiple purpose.

### Pattern 3: Multiple contexts

Using multiple contexts is fine and rather recommended to separate concerns.
Contexts don't have to be application-wide and they can be used
for parts of component tree. Only if your contexts can be used anywhere
in your app, defining them at the root is a good reason.

```jsx
const ContextA = createContext(null);
const ContextB = createContext(null);
const ContextC = createContext(null);

const App = () => {
  const [stateA, dispatchA] = useReducer(reducerA, initialStateA);
  const [stateB, dispatchB] = useReducer(reducerB, initialStateB);
  const [stateC, dispatchC] = useReducer(reducerC, initialStateC);
  const valueA = useMemo(() => [stateA, dispatchA], [stateA]);
  const valueB = useMemo(() => [stateB, dispatchB], [stateB]);
  const valueC = useMemo(() => [stateC, dispatchC], [stateC]);
  return (
    <ContextA.Provider value={valueA}>
      <ContextB.Provider value={valueB}>
        <ContextC.Provider value={valueC}>
          ...
        </ContextC.Provider>
      </ContextB.Provider>
    </ContextA.Provider>
  );
};

const Component1 = () => {
  const [stateA, dispatchA] = useContext(ContextA);
  return (
    ...
  );
};
```

This is going to be a bit of mess, if we have more contexts.
It's time to introduce some libraries.
There are several libraries to support multiple contexts
and some of them provide hooks API.

----

I've been developing such a library called "react-hooks-global-state".

https://github.com/dai-shi/react-hooks-global-state

Here's example code how it looks like.

```jsx
import { createGlobalState } from 'react-hooks-global-state';

const initialState = { 
  a: ...,
  b: ...,
  c: ...,
};
const { GlobalStateProvider, useGlobalState } = createGlobalState(initialState);

const App = () => (
  <GlobalStateProvider>
    ...
  </GlobalStateProvider>
);

const Component1 = () => {
  const [valueA, updateA] = useGlobalState('a');
  return (
    ...
  );
};
```

There is at least a caveat in this library.
It uses an undocumented feature called `observedBits` and not only
it's unstable, but with its limitation, this library is only performant
if the number of sub-states (like `a`, `b`, `c`) is equal or smaller than 31.

### Pattern 4: Redux

The biggest limitation with multiple contexts is that
dispatch functions are also separated.
If your app gets big and several contexts need to be updated
with a single action, it's time to introduce Redux.
(Or, actually you could dispatch multiple actions for a single event,
I personally don't like that pattern very much.)

There are various libraries to use Redux with hooks, and
the official react-redux is about to release its hooks API.

----

Since I've put lots of effort in this domain,
let me introduce my library called "reactive-react-redux".

https://github.com/dai-shi/reactive-react-redux

Unlike traditional react-redux, this library doesn't
require `mapStateToProps` or a selector.
You can simply use global state from Redux
and the library tracks the state usage with Proxy for optimization.

Here's example code how it looks like.

```jsx
import { createStore } from 'redux';
import {
  ReduxProvider,
  useReduxDispatch,
  useReduxState,
} from 'reactive-react-redux';

const initialState = {
  a: ...,
  b: ...,
  c: ...,
};

const reducer = (state = initialState, action) => {
  ...
};

const store = createStore(reducer);

const App = () => (
  <ReduxProvider store={store}>
    ...
  </ReduxProvider>
);

const Component1 = () => {
  const { a } = useReduxState();
  const dispatch = useReduxDispatch();
  return (
    ...
  );
};
```

### Final thoughts

For moderate to large apps, it is likely that a single event
changes several parts of the state and hence the UI.
So, the use of Redux (or any kind of app state management)
seems natural in this case.

However, apollo-client and forthcoming react-cache would
play a role of data management, and the role of UI state management
would get smaller. In that case, the multiple contexts pattern
could make more sense for moderate apps.
