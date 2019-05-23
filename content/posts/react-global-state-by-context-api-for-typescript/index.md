---
title: "React global state by Context API for TypeScript"
subtitle: "A tiny library to help storing global state in React for both JavaScript and TypeScript."
date: 2018-11-08T12:00:00+09:00
tags: ["react", "context", "global-state"]
---

_Note: this post has nothing to do with React Hooks API._

In [my previous post](https://blog.axlight.com/posts/react-global-state-by-context-api/), I introduced a library called react-context-global-state. After a while, I have better knowlege of TypeScript (yes, I'm a beginner!), and come up with fairly good type definitions. I just released a new version.

https://www.npmjs.com/package/react-context-global-state

### Example code in TypeScript

Let me breifly show how the code looks like in TypeScript.

In a file named state.ts, we define a global state and export function components. The `State` type is also exported.

```javascript
import { createGlobalState } from 'react-context-global-state';

const initialState = {
  counter1: 0,
  person: {
    age: 0,
    firstName: '',
    lastName: '',
  },
};

export type State = typeof initialState;

export const { StateProvider, StateConsumer } = createGlobalState(initialState);
```

In App.tsx, we provide the state using Context.

```jsx
import * as React from 'react';

import { StateProvider } from './state';

import Counter from './Counter';

const App = () => (
  <StateProvider>
    <h1>Counter</h1>
    <Counter />
    <Counter />
  </StateProvider>
);

export default App;
```

In Counter.tsx, we consume the part of the state by specifying the name.

```jsx
import * as React from 'react';

import { StateConsumerType } from 'react-context-global-state';

import { State, StateConsumer } from './state';

const Counter1StateConsumer = StateConsumer as StateConsumerType<State, 'counter1'>;

const Counter = () => (
  <Counter1StateConsumer name="counter1">
    {(value, update) => (
      <div>
        <span>
          Count:
          {value}
        </span>
        <button type="button" onClick={() => update(v => v + 1)}>+1</button>
        <button type="button" onClick={() => update(v => v - 1)}>-1</button>
      </div>
    )}
  </Counter1StateConsumer>
);

export default Counter;
```

Notice how `Counter1StateConsumer` is defined above. I'm not quite sure if this is the best way in TypeScript. Probably, it's ideal if it could just be constrained by the `name="counter1"` prop.

The full example code is available in the repository.

https://github.com/dai-shi/react-context-global-state

And, you can run it like the following and open http://localhost:8080 in your browser to see it working.

```bash
git glone https://github.com/dai-shi/react-context-global-state.git
npm install
npm run examples:typescript
```

### Some thoughts
While global state is convenient, it's better not to overuse it. It's essentially a global variable, which in general should be avoided. In React, we should first consider using local state and passing it in the component tree. Global state is certainly useful if two components far apart in the tree need to share a value. In such a case, the Context API may fit well. This library is meant for rather small applications in which using Redux or alike requires too much effort.
