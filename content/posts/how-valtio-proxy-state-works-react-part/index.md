---
title: "How Valtio Proxy State Works (React Part)"
subtitle: "useSyncExternalStore and proxy-compare"
date: 2021-12-26T22:00:00+09:00
tags: ["react", "global-state", "proxy"]
---

## Introduction

In [the previous article](https://blog.axlight.com/posts/how-valtio-proxy-state-works-vanilla-part/),
we explained how [Valtio](http://github.com/pmndrs/valtio)
proxy state works.
It tracks mutations of state and create immutable snapshot.

Let's recap the API in vanilla part of Valtio.

```js
// Create a new proxy state to detect mutations
const state = proxy({ count: 0 });

// You can mutate it
++state.count;

// Create a snapshot
const snap1 = snapshot(state); // ---> { count: 1 }

// Mutate it again
state.count *= 10;

// Create a snapshot again
const snap2 = snapshot(state); // ---> { count: 10 }

// The previous snapshot is not changed
console.log(snap1); // ---> { count: 1 }

// You can subscribe to it
subscribe(state, () => {
  console.log('State changed to', state);
});

// Then, mutate it again
state.text = 'hello'; // ---> "State changed to { count: 10, text: 'hello' }"
```

Now, let's see how we can use the state in React.

## Introducing useSyncExternalStore

React 18 provides a new hook called `useSyncExternalStore`.
It's designed to safely use an external store in React.
Our proxy object in Valtio is exactly an external store.

As we have `snapshot` function to create immutable state,
it should be pretty straightforward.

```js
// Create a state
const stateFoo = proxy({ count: 0, text: 'hello' });

// Define subscribe function for stateFoo
const subscribeFoo = (callback) => subscribe(stateFoo, callback);

// Define snapshot function for stateFoo
const snapshotFoo = () => snapshot(stateFoo);

// Our hook to use stateFoo
const useStateFoo = () => useSyncExternalStore(
  subscribeFoo,
  snapshotFoo
);
```

How simple! We could build a custom hook to handle any proxy state.
We just need not to forget to use `useCallback`.

But, Valtio has a more advanced feature,
automatic render optimization.

## What is automatic render optimization

Render optimization is to avoid extra re-renders,
which produce the results that makes no difference to users.
In the case of `stateFoo`, suppose we have a component that shows
the `text` value in `stateFoo`.

```jsx
const TextComponent = () => {
  const { text } = useStateFoo();
  return <span>{text}</span>;
};
```

If we change the `count` value in `stateFoo`, like `++stateFoo.count`,
this `TextComponent` actually re-renders,
but produces the same result because it doesn't use
the `count` value, and the `text` value isn't changed.
So, this is an extra re-render.

Render optimization is to avoid such extra re-renders,
and one way to solve it is to manually tell the hook
which properties we will be using.

For example, if we assume the hook accepts a list of strings,
we would be able to tell the properties like the following.

```jsx
const TextComponent = () => {
  const { text } = useStateFoo(['text']);
  return <span>{text}</span>;
};
```

Automatic render optimization is to do this automatically.
Is this possible?
It's possible with utilizing proxies.
Proxies allow us to detect state property access.
I have been working on this for years, and
[react-tracked](https://react-tracked.js.org)
is one of the resulting projects that use this technique.
We have an internal library called proxy-compare.

## How proxy-compare works

[proxy-compare](https://github.com/dai-shi/proxy-compare)
is a library to enable automatic render optimization.

What we would like to know is, in the previous example,
the `text` value is used in `TextComponent`.

Let's see how it can be done with proxies.

```js
// An array to store accessed properties
const accessedProperties = [];

// Wrap stateFoo with Proxy
const obj = new Proxy(stateFoo, {
  get: (target, property) => {
    accessedProperties.push(property);
    return target[property];
  },
});

// Use it
console.log(obj.text);

// We know what are accessed.
console.log(accessedProperties); // ---> ['text']
```

That's the basic idea.
To extend it, we want to support accessing nested objects.

```js
// More complex state
const obj = { nested: { count: 0, text: 'hello' }, others: [] };

// Use a nested property
console.log(obj.nested.count);

// As a result, `nested.count` is detected as used.
// `nested.text` and `others` are known to be unused.
```

This is quite a bit of task, but proxy-compare handles such cases.
And, it's done in a pretty efficient way.
If you are curious, check out the source code of proxy-compare.

Valtio provides a hook based on proxy-compare
to enable automatic render optimization.

## Valtio's solution: useSnapshot

The hook provided by Valtio is called `useSnapshot`.
It returns an immutable snapshot, but it's wrapped with
proxies for render optimization.

It can be used like the following.

```jsx
import { proxy, useSnapshot } from 'valtio';

const state = proxy({ nested: { count: 0, text: 'hello' }, others: [] });

const TextComponent = () => {
  const snap = useSnapshot(state);
  return <span>{snap.nested.text}</span>;
};
```

This component only re-renders when the `text` value is changed.
Even if `count` or `others` changes, it won't re-render.

The implementation of `useSnapshot` is a little bit tricky,
and we don't dive in deep.
Basically, it's just a combination of `useSyncExternalStore`
and `proxy-compare`.

Valtio's mutable state model matches pretty well with
the mental model of `useSnapshot`.
You basically define a state object with `proxy`,
use it with `useSnapshot` and you can mutate
the state object as you like.
The library takes care of everything else.

To be fair, there are some limitations due to how proxies work.
For example, proxies can't detect mutations on `Map`.
Another example is that proxies can't detect a use of `Object.keys`.

## Closing note

Hopefully, we explained the overall concept of Valtio
with [the previous article](https://blog.axlight.com/posts/how-valtio-proxy-state-works-vanilla-part/) and this one.
The actually implementation has some more work
to handle some edge cases and for efficiency.
Having that said, we think it's fairly small
and we encourage people with some interests to have a read.

<https://github.com/pmndrs/valtio>
