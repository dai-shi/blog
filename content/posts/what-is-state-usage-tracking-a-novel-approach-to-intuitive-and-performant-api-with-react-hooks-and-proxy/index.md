---
title: "What is state usage tracking? A novel approach to intuitive and performant global state with React hooks and Proxy"
subtitle: "For both Redux and non-Redux"
date: 2019-07-07T17:00:00+09:00
tags: ["react", "hooks", "redux", "proxy", "global-state"]
---

### Introduction

There are many libraries for global state with React hooks.
React Redux also provides hooks API, which is very clean.

In general, I would avoid using global state.
It would reduce the isolation of components.
Multiple contexts should work fine for certain use cases.

But, what if we really need a global state.

### Problem

When a state is a non-trivial object, it's not likely to
use all properties of the object for one component to render. 
What most libraries do is to provide a selector interface.
With the selector interface,
developers can specify which part of the state to use in a component.
In general, a selector is a function,
but there are alternative ways to specify part of the state.
For example, by property names or paths.
In any case, developers are responsible to write proper selectors. 

This is not only about React Redux, but applicable to most libraries. 

### Solution "state usage tracking"

State usage tracking is to automate that process.
Instead of developers specify which part of a state to be used,
the system tracks how the state is used.
Proxy API play the role of tracking.
The idea to use Proxy API for tracking is not new.
Immer and MobX use Proxy to detect changes.
The difference is the purpose.
Immer uses Proxy to detect mutation or say "write operation."
Whereas, state usage tracking is for "read operation."

My proposition is to combine React's reactive system with Proxy-based tracking. 
Thanks to React hooks, it's extremely easy to use.
My current implementation provides `useTrackedState` hook.
If you call this hook in render, you get a state back.
You can then use the state in render.
The hook automatically tracks the usage of the state in render.
With tracking, the hook will only trigger re-render
if the used part of the state is changed.
Because there's no point of re-rendering
if only unused part of the state is changed. 

### No semantics change

It's important to note that state usage tracking won't change any semantics.
Let's suppose only unused part of the state is changed.
In this case, the hook triggers re-render,
but a component will render the correct result. 
If the hook doesn't actually track anything,
we will get the same result.
The difference is only that it may slow down. 

The point is that there is no semantics change in the useTrackedState hook.
It only optimizes re-renders. Developers need to code what, not how.
It's different from using selectors to control re-renders.

### Performance

Only the remaining question is the optimization in practice,
because it comes at a cost. That is why benchmarking is important.
The hook is simple and straightforward to use.
If it's usable with comparable performance, it's good to go.

[See this post for the benchmark result](https://blog.axlight.com/posts/benchmark-react-tracked/)

### Projects using state usage tracking

- [react-tracked](https://github.com/dai-shi/react-tracked): Non-Redux global state
- [reactive-react-redux](https://github.com/dai-shi/reactive-react-redux): React Redux alternative

### Closing notes

This short post explained the idea of state usage tracking.
Unlike my other posts, there was no code snippet.
I hope the idea is explained well without code.
I appreciate for any feedback so that I could write a follow-up post.

