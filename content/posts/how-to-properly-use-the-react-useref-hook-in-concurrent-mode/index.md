---
title: "How to properly use the React useRef hook in Concurrent Mode"
subtitle: "Concurrent Mode requires stricter way of writing components."
date: 2019-03-07T12:00:00+09:00
tags: ["react", "hooks", "concurrent"]
---

### Introduction

According to [React 16.x Roadmap](https://reactjs.org/blog/2018/11/27/react-16-roadmap.html), we are expecting Concurrent Mode soon.

> React 16.x (~Q2 2019): The One with Concurrent Mode
> Concurrent Mode lets React apps be more responsive by rendering component trees without blocking the main thread. It is opt-in and allows React to interrupt a long-running render (for example, rendering a new feed story) to handle a high-priority event (for example, text input or hover). Concurrent Mode also improves the user experience of Suspense by skipping unnecessary loading states on fast connections.

While it is an opt-in feature, you can enable it easily and if your component isn't properly implemented, it won't work correctly. In short, you can't make side effects in your render function. This has always been true, but it hadn't been a real issue until we have Concurrent Mode. In Concurrent Mode, render functions could be invoked multiple times without actually committing (meaning, for example, applying changes to the DOM). Luckily, Strict Mode intentionally invoke render functions twice, and you can notice the wrong behavior in the development mode. Refer [the doc](https://reactjs.org/docs/strict-mode.html#detecting-unexpected-side-effects) for more information.

This short article focuses on `useRef`, one of React Hooks. The useRef hook is pretty powerful and often can be abused. In general, developers should avoid using `useRef` if they could use `useState` instead.

This article shows example code that uses `useRef` improperly and how to fix it. The example is a simple counter just to illustrate the issue. It's not product code, and you can actually implement the same example with `useState`.

### The bad code

```jsx
import React, { useRef } from "react";
const BadCounter = () => {
  const count = useRef(0);
  count.current += 1;
  return <div>count:{count.current}</div>;
};
export default BadCounter;
```

It works as expected in a traditional React where the render phase and the commit phase is one-to-one. However, if it invokes the render function multiple time without committing, the count increases unexpectedly.

### The good code

```jsx
import React, { useEffect, useRef } from "react";
const GoodCounter = () => {
  const count = useRef(0);
  let currentCount = count.current;
  useEffect(() => {
    count.current = currentCount;
  });
  currentCount += 1;
  return <div>count:{currentCount}</div>;
};
export default GoodCounter;
```

This code uses `useEffect`, whose first argument function is only invoked in the commit phase. The `currentCount` is a local variable within the render function scope, and it will only change the ref `count` in the commit phase. The ref is essentially a global variable outside the function scope, hence modifying it is a side effect.

### The demo

To run the two code examples above, here's the App component.

```jsx
import React, {
  useReducer,
  unstable_ConcurrentMode as ConcurrentMode
} from "react";
import BadCounter from "./BadCounter";
import GoodCounter from "./GoodCounter";
const useForceUpdate = () => useReducer(state => !state, false)[1];
const App = () => {
  const forceUpdate = useForceUpdate();
  return (
    <ConcurrentMode>
      <div>
        <button onClick={forceUpdate}>Update</button>
        <h3>Bad Counter</h3>
        <BadCounter />
        <h3>Good Counter</h3>
        <GoodCounter />
      </div>
    </ConcurrentMode>
  );
};
export default App;
```

(In fact, we don't need `ConcurrentMode`, but just `StrictMode` is enough.)

Please checkout the following codesandbox to see the actual behavior.

<iframe src="https://codesandbox.io/embed/x2p46v02z4?fontsize=14" title="x2p46v02z4" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

### Final notes

The reason I want to use `useRef` is to develop a bindings library for Redux. It requires to subscribe the global store and update components when the state is updated. The ref is used to keep track of the last rendered state. For more information, check out the GitHub repository.

https://github.com/dai-shi/react-hooks-easy-redux
