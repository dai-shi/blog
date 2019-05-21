---
title: "TypeScript-aware React hooks for global state"
date: 2018-10-28T12:00:00+09:00
tags: ["react", "hooks", "typescript", "global-state"]
---

##### React hooks are awesome.

React Hooks API is a new proposal for React v16.7. It enables writing all components as functions with full functionality. You can develop "custom hooks" to add a new functionality. Although it's not released yet, I would like to introduce a new library to support global state. Note that it's still an experiment.

### The library

It's called "react-hooks-global-state" and is available in npm. Please also checkout the GitHub repository.

https://github.com/dai-shi/react-hooks-global-state

### How to code with it

First, you need to define a global state and export a Provider and a custom hook.

```javascript
import { createGlobalState } from 'react-hooks-global-state';

export const { GlobalStateProvider, useGlobalState } = createGlobalState({
  counter: 0,
  person: {
    age: 0,
    firstName: '',
    lastName: '',
  },
});
```

Then, use the hook to implement a Component.

```jsx
import * as React from 'react';

import { useGlobalState } from './state';

const Counter = () => {
  const [value, update] = useGlobalState('counter');
  return (
    <div>
      <span>
        Count:
        {value}
      </span>
      <button type="button" onClick={() => update(v => v + 1)}>+1</button>
      <button type="button" onClick={() => update(v => v - 1)}>-1</button>
    </div>
  );
};

export default Counter;
```

Even if you use the `Counter` component twice, they share the state.

```jsx
import * as React from 'react';

import { GlobalStateProvider } from './state';
import Counter from './Counter';
import Person from './Person'; // Refer the code in the repository

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

The full code is in `examples/02_typescript` folder in the repository. You can try this example by cloning the repository and run:

```bash
git clone https://github.com/dai-shi/react-hooks-global-state.git
npm install
npm run examples:typescript
```

Open http://localhost:8080 after it's up and running.

### Is this TypeScript?

Actually, yes! It's fully typed, and furthermore there's no implicit any type in the code. The example code is written in TypeScript, but the library itself is written in JavaScript with type definitions, so you can use it either way as you like.

### Any feedbacks?

Feedbacks are appreciated via Twitter or GitHub issues. Thanks!

### Changelog

- _[2018-10-28]_: Initial publication.
- _[2018-11-08]_: Follow the library API change and make the example more intuitive.
- _[2018-11-12]_: The library API is changed again based on the Context API.
