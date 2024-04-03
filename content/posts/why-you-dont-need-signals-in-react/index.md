---
title: "Why You Don't Need Signals in React"
subtitle: "Because we have Jotai"
date: 2023-04-23T21:00:00+09:00
tags: ["react", "hooks", "global-state", "atom", "jotai", "signals"]
---

### Introduction

In the world of web frontend development, signals have become a popular topic.
At their core, signals are used to represent changes in state over time.
Some developers have discussed the potential of using signals
in conjunction with React.

Signals are actually an older concept, and it's uncertain
how they are understood by modern web developers.
Initially, I was confused about the characteristics of signals,
but I later realized that they can be boiled down to two main aspects:

- a) Reactive primitives
- b) Bypassing diffing

{{< tweet user="dai_shi" id="1628926485200523264" >}}

In this blog post, we'll explore these two aspects
and their relevance in the context of React.
Note that we'll only be discussing these two aspects of signals
and won't consider other potential uses
which might be more important for someone.

### Reactive primitives

Reactivity is a key feature of React.
Components are re-rendered when state changes.
With `useState`, you can create reactive primitives by defining state
that triggers re-rendering on updates.
This makes your components reactive.
Additionally, you can define derived state in render functions.

Here's an example usage of `useState`:

```js
const Component = () => {
  const [count, setCount] = useState(0);
  const doubleCount = count * 2; // derived state
  // ...
};
```

However, it's important to note that `useState` only creates local state.
This can make it difficult to share state between components,
and you may need to use prop drilling or context propagation to accomplish this.

To simplify the process of defining and using global state,
third-party libraries like [Jotai](https://jotai.org) can be useful.
With Jotai, you can easily share state between components
without relying on prop drilling or context propagation.

To define global state with Jotai, you can use atoms.
These atoms represent definitions of pieces of state
that you can use in your components.
For example, here's how you can define a primitive atom:

```js
const countAtom = atom(0);
```

You can also define derived atoms that depend on other atoms, like this:

```js
const doubleCountAtom = atom((get) => get(countAtom) * 2);
```

While the syntax of atoms may look a bit different from typical signal syntax,
the mental model is quite similar.
We define primitives and compose them for complex state.
You can define as many atoms as you need to represent a data graph,
making it easy to manage complex state in your application.

The following shows how to use Jotai atoms in your components:

```js
const Component = () => {
  const [count, setCount] = useAtom(countAtom);
  const [doubleCount] = useAtom(doubleCountAtom); // derived state
  // ...
};
```

Unlike `useState`, `useAtom` is not local state and
you can use it in another component to share the atom state:


```js
const AnotherComponent = () => {
  const [count, setCount] = useAtom(countAtom);
  // ...
};
```

You may have noticed that Jotai atoms work similarly to signals
when it comes to reactive primitives.
However, Jotai offers additional benefits through React hooks like `useAtom`,
which follow React conventions.
These hooks allow you to share state between components
without the need for prop drilling or context propagation,
simplifying your code and making it easier to reason about.
Moreover, Jotai has `Provider` to isolate state for subtrees,
which is not possible with truly global signals.

### Bypassing diffing

Another key feature of React is updating of the DOM
achieved through a process called "diffing."
By comparing the previous and current representations of your UI,
React determines what has changed and updates only those parts of the DOM,
resulting in better performance and a more responsive UI.

However, diffing does come at a cost,
and there may be cases where bypassing diffing can be more efficient.
For example, if you're updating only one part of the UI
and are certain that everything else is unchanged,
updating the UI directly without diffing may be more efficient.

To demonstrate that it's technically possible to bypass diffing,
we have an experimental library called
[jotai-signal](https://github.com/jotaijs/jotai-signal).
We also have a blog post discussing its internals:
[Demystifying Create React Signals Internals](https://blog.axlight.com/posts/demystifying-create-react-signals-internals/)

However, bypassing diffing goes against React's core principles
of declarative programming.
While it may be tempting to bypass diffing for performance reasons,
doing so risks introducing inconsistencies in your UI and
making your application harder to reason about.

Before deciding to bypass diffing, it's important to thoroughly
assess the performance benefits and weigh them against the potential risks.
It's also worth considering whether there are other ways to
optimize your application's performance.

In general, it's recommended to follow React's best practices
and use diffing appropriately
to ensure that your UI remains consistent and predictable.

### Summary

In conclusion, while signals are an interesting concept in web development,
Jotai offers a simpler and more intuitive way to manage state
in React applications.
With Jotai, you can easily create and use global state through atoms,
eliminating the need for prop drilling or context propagation.
Atoms are conceptually very similar to signals in terms of
reactive primitives.

While it's technically possible to
bypass React's diffing algorithm in a sense,
doing so goes against the principles of declarative programming
and can introduce inconsistencies in your UI.
It's important to thoroughly assess the potential benefits
and risks before making the decision to bypass diffing.

By following React's best practices and leveraging the power of Jotai,
you can create maintainable and performant React applications.
