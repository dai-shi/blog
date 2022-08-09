---
title: "An alternative to React Redux by React Hooks API (For both JavaScript and TypeScript)"
subtitle: "React hooks are awesome."
date: 2018-11-04T12:00:00+09:00
tags: ["react", "hooks", "typescript", "global-state", "redux"]
---

### Motivation

Everyone is excited about the new React Hooks API. So am I. Having been thinking how to manage global state, the Hooks API seems promising. By the way, I like Redux a lot, but I don't like react-redux a.k.a `connect` very much. It is too complicated for beginners to use it properly. For example, reselect / memoization is a hard concept to explain. My recommendation is to structure a global state so that `mapStateToProps` only needs to select a part of the state without any logic. If you are really free to structure a global state, you can make it so that it selects direct properties of the state, which means "one-depth".

### My library

With that in mind, I've been developing a library for managing global state.

https://github.com/dai-shi/react-hooks-global-state

The global state in this library is an object which consists of items. For example:

```javascript
const state = {
  name: 'foo',
  age: 23,
  hobbies: ['reading', 'videogaming'],
  scores: { stageA: 3, stageB: 7, stageC: 2 },
};
```

This state consists of 4 items (name, age, hobbies and scores). Each item has a value, which can be not only a primitive value but also an array or an object. You can connect to each item to get notified when a value is updated, but not a deep value in the object tree.

This library recently supports a reducer for updating states. So, you might expect it as a replacement for Redux for React. Maybe, maybe not. You will see it by yourselves.

### Example code

You define a global state and export some methods like the following.

```javascript
import { createStore } from 'react-hooks-global-state';

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setFirstName', firstName: string }
  | { type: 'setLastName', lastName: string }
  | { type: 'setAge', age: number };

export const { GlobalStateProvider, dispatch, useGlobalState } = createStore(
  (state, action: Action) => {
    switch (action.type) {
      case 'increment': return {
        ...state,
        counter: state.counter + 1,
      };
      case 'decrement': return {
        ...state,
        counter: state.counter - 1,
      };
      case 'setFirstName': return {
        ...state,
        person: {
          ...state.person,
          firstName: action.firstName,
        },
      };
      case 'setLastName': return {
        ...state,
        person: {
          ...state.person,
          lastName: action.lastName,
        },
      };
      case 'setAge': return {
        ...state,
        person: {
          ...state.person,
          age: action.age,
        },
      };
      default: return state;
    }
  },
  {
    counter: 0,
    person: {
      age: 0,
      firstName: '',
      lastName: '',
    },
  },
);
```

Don't be bothered by the `Action` type if you are not familiar with TypeScript. The first argument of `createStore` is a reducer, which should look familiar if you have ever used Redux. A required initial global state is passed to the second argument. You could use `combineReducers` if you like. Notice the result of `createStore` is exported.

You need to wrap your top-level component with `GlobalStateProvider` to attach a context that holds a global state.

```jsx
import * as React from 'react';

import { GlobalStateProvider } from './state';

import Counter from './Counter';
import Person from './Person';

const App = () => (
  <GlobalStateProvider>
    <h1>Counter</h1>
    <Counter />
    <Counter />
    <h1>Person</h1>
    <Person />
    <Person />
  </GlobalStateProvider>
);

export default App;
```

The following is how to use the counter in a component.

```jsx
import * as React from 'react';
import { dispatch, useGlobalState } from './state';

const increment = () => dispatch({ type: 'increment' });
const decrement = () => dispatch({ type: 'decrement' });

const Counter = () => {
  const [value] = useGlobalState('counter');
  return (
    <div>
      <span>Count: {value}</span>
      <button type="button" onClick={increment}>+1</button>
      <button type="button" onClick={decrement}>-1</button>
    </div>
  );
};
export default Counter;
```

This should be easy enough. The `dispatch` and `useGlobalState` is imported from the previous file. Two callbacks are defined and use `dispatch`. You might be wondering why we have `dispatch` somewhat globally. It certainly reduces the code isolation, but this allows defining actions outside of components. Please correct me if I misunderstand something. We could also provide `dispatch` in another context and something like `useDispatch`.

The rest of the code including the Person component can be found in the repository: https://github.com/dai-shi/react-hooks-global-state/tree/master/examples/06_reducer

You can run the example by:

```bash
git clone https://github.com/dai-shi/react-hooks-global-state.git
npm install
npm run examples:reducer
```

Then, open http://localhost:8080 in the browser.

### Important Notes

React Hooks API is not yet released and this library is still an experiment at this point. Feedbacks are welcomed to improve the library. I wonder many people are trying the similar thing and I want to get ideas if possible.

### Changelog

- _[2018-11-04]:_ Initial publication.
- _[2018-11-08]:_ Follow the library API change and make the example more intuitive.
- _[2018-11-12]:_ The library API is changed again based on the Context API.
