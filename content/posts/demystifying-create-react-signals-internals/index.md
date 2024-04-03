---
title: "Demystifying Create React Signals Internals"
subtitle: "How jotai-signal, valtio-signal and zustand-signal work"
date: 2023-02-26T23:00:00+09:00
tags: ["react", "signals", "global-state"]
---

## Introduction

When I first saw SolidJS and Preact Signals,
I thought they are interesting but they are different from React.
What made me motivated is [@preact/signals-react](https://www.npmjs.com/package/@preact/signals-react).
I didn't like the original implementation using REACT INTERNALS,
and that drove me to create something.

We already had Jotai. Jotai atoms are like signals,
representing reactive sources,
and allowing to form a dependency graph or a data flow.
The missing piece was signal-like syntax without hooks.

The initial version of [jotai-signal](https://github.com/jotaijs/jotai-signal)
library was developed as a proof of concept.
It provided a custom JSX transfomer to hide hooks behind the scene.
It was using `experimentl_use` to read React context.

I also developed [jotai-uncontrolled](https://github.com/jotaijs/jotai-uncontrolled) to bypass diffing,
which is another aspect of signals.
While `jotai-signal` was a syntax sugar, `jotai-uncontrolled`
has a performance benefit.

Meanwhile, the next version of `jotai-signal` uses Jotai v2 API,
and avoids `experimental_use` by exposing Store interface.
This allows us to consider using it more seriously.
Furthermore, when discussing internally, I got an idea to combine
`jotai-signal` and `jotai-uncontrolled`.
We can bypass diffing automatically in some cases.

Along with it, I developed `valtio-signal` and `zustand-signal`
by almost copying code from `jotai-signal`.
This made me think it would be possible to create
an abstract layer, and `create-react-signals` was born.

This post will explain how `create-react-signals` work internally.

## What are signals?

In the context of `create-react-signals`,
a signal is kind of a store that has three functions:

- `subscribe`: a function to add a callback which will be invoked when a signal value changes.
- `getValue`: a function to return the signal value.
- `setValue`: a function to update the signal value.

Using a signal with a custom hook should be pretty easy.

```js
const [value, setValue] = useSignal(signal);
```

You can just subscribe in the hooks and return the current value.
Implementation should be trivial with `useSyncExternalStore`.

But, we want to just use the signal
so that it automatically subscribes and updates value.

It could be done with a compiler at build time,
but our approach is a runtime transformation.

## How does it transform code?

Let's assume we have a simple signal.
Suppose `countSignal` contains a number.

Our component would look like this:

```jsx
const Component = () => {
  return (
    <div>{countSignal}</div>
  );
};
```

React doesn't understand this, because `countSignal` is a special object
that can't be rendered.

We transform the code into something like the following.

```jsx
const SignalsRerenderer = ({ signal, render }) => {
  const [, rerender] = useReducer((c) => c + 1), 0);
  useEffect(() => signal.subscribe(), [signal]);
  return render();
};

const Component = () => {
  return SignalsRerenderer({
    signal: countSignal,
    render: () => (
      <div>{countSignal.getValue()}</div>
    ),
  });
};
```

`.subscribe`. and `.getValue` methods are pseudo code,
as a signal doesn't explicitly have such methods.
There are other simplifications, like only handling one signal
and omitting memoization code.

To implement the transformation,
`create-react-signals` creates a custom `createElement` from
original `React.createElement`.

Overriding the original `React.createElement` might be
one solution, but it seems too hacky.

## Why custom JSX transfomer?

Forunately, we can customize JSX transfomer.
The recent bundler supports `@jsxImportSource`.

- https://babeljs.io/docs/babel-preset-react#importsource
- https://www.typescriptlang.org/tsconfig#jsxImportSource

You can also specify a pragma in your source code.

```js
/** @jsxImportSource jotai-signal */
```

This technique is used in some projects, for example,
[Emotion](https://emotion.sh/docs/css-prop).

One of the biggest questions in this approach is
that specifying the custom transformer can be seen unusual.
(Well, it's a hack after all, so it's not usual.)

## How can signals create signals?

What if a signal value is an object?
Suppose `personSignal` has a value `{ firstName: 'first', lastName: 'last' }`.

Using an object signal in JSX like the following doesn't work.

```jsx
const Component = () => {
  return (
    <div>{personSignal}</div>
  );
};
```

As an object can't be rendered, we need to do something like the following.

```jsx
const Component = () => {
  return (
    <div>{personSignal.firstName}</div>
  );
};
```

Now, to make that work, `personSignal.firstName` shouldn't be a string.
It has to be another signal. Otherwise, we can't subscribe to it.

How do we solve it?
Our current solution is Proxies.
When there's a property access to a signal,
it will create a new signal.
This is done recursively.

If `personSignal.firstName` is used in JSX,
it will skip updating when only `lastName` changes.

(Note that object signals are currently not supported in `jotai-signal`.)

## How does it skip diffing?

As noted previously, `jotai-uncontrolled` allows skipping diffing.
It works like [uncontrolled components](https://reactjs.org/docs/uncontrolled-components.html).
The technique is using `ref` and manipulating DOM directly.

This post doesn't go too much in details
about the implementation of uncontrolled components.

There's fallback mechanism if uncontrolled components don't work.

{{< tweet user="dai_shi" id="1616639032418799619" >}}

This approach is still considered experimental
and might have a pitfall.

## What are difficulties?

One of unsolved issues is
how to get signal values outside JSX.
Our signal has `getValue` function, so we have to call it.

There are some options:

```js
// callable signal
const value = countSignal();

// string property access
const value = countSignal.value;

// symbol property access
const value = countSignal[VALUE_SYMBOL];
```

So, how is it difficult?
Consider the `personSignal` case.

```js
// callable signal
const value = personSignal.firstName();

// string property access
const value = personSignal.firstName.value;

// symbol property access
const value = personSignal.firstName[VALUE_SYMBOL];
```

At first, the callable signal style looked the best,
but it can't be typed in TypeScript.

The string property looks the most familiar,
but it can cause naming conflict.

The symbol property has no drawbacks, but not very handy.

The same problem applies to `setValue`.

```js
// callable signal
const value = personSignal.firstName('newValue');

// string property access
const value = personSignal.firstName.value = 'newValue';

// symbol property access
const value = personSignal.firstName[VALUE_SYMBOL] = 'newValue';
```

We could change our mind and use a property to return a signal.
```js
const countValue = countContainer.value;
const countSignal = countContainer.signal;
```

In summary, it's still an open problem.

## Closing notes

In addition to `jotai-signal`, `valtio-signal` and `zustand-signal`,
we can technically create `redux-signal`.

{{< tweet user="dai_shi" id="1629482966895448064" >}}

I think signals in React are still a open research field.

If React will provide a new primitive such as `use(Observable)`,
we could explore this approach further.

{{< tweet user="dai_shi" id="1628951555658620930" >}}

Until then, let's play with userland solutions.
