---
title: "How Valtio Was Born"
subtitle: "Did we need the third library?"
date: 2024-08-18T21:00:00+09:00
tags: ["react", "global-state", "typescript", "valtio"]
---

### Introduction

There was a discussion in our team after releasing Zustand v3 and the brand new Jotai. It was about whether we could develop another library for global state.

In this post, I will reflect on the start of Valtio's development and its API design.

### Hesitation at First

While the idea of a proxy state sounded promising, I was hesitant to develop a third global state library at first. We were already maintaining two libraries in the same problem domain. It felt like adding another library would make no sense.

Another concern was that MobX had existed for quite a while, and it wasn't certain if we could compete with it. I especially wasn't sure what could be a fundamental difference.

### The Challenge to Solve

There was a challenge with using mutable state in React. The React contract is based on immutability, and simply using mutable state doesn't work well with React, especially with concurrency. Without immutable state, the hooks API is infeasible. I had known about this issue previously while developing React Tracked and had even suggested a potential solution to the MobX team for their possible hooks API. Unfortunately, they didn't take that path and decided to stick with the HoC API.

This made me think that if I could solve the problem and provide a hooks API for mutable state, it could be a big difference from MobX.

The goal was simple: support `state.count++` syntax and provide a hook for React.

### Another Motivation

As noted, I had developed a library called React Tracked.

<https://react-tracked.js.org>

It's somewhat used in production, but the use of Proxies prevented some users from adopting it. Some preferred an alternative internal library, `use-context-selector`, which doesn't use Proxies.

If I were to create a new library with a proxy state, it might be a good chance to provide the same capability as React Tracked. Since it's already proxy-based, the use of Proxies isn't a hurdle. Luckily, I had already developed an internal library called `proxy-compare`, specifically for React Tracked. It's a fundamental library for tracking state usage and could be used in the new library.

### The Birth of Valtio

My objective was to develop a library that would support mutable state with a hooks API, utilizing my internal library `proxy-compare`. The key idea was "snapshot." To make it work with React hooks, we needed to create an immutable object from a mutable state. I call it a snapshot. With the immutable object, `proxy-compare` works flawlessly.

My first prototype was developed within a day or two. The API was very simple, and everything worked behind the scenes. I was very excited about the outcome.

<https://github.com/pmndrs/valtio>

### The API Design of Valtio

Speaking of the API, our design philosophy is to make it feel like just JavaScript. We wanted `state.count++` to work, meaning you can read a value with `state.count` and write with `state.count = 1`. Basically, you wouldn't notice if it's a proxy or not.

So, APIs like `state.count.get()` and `state.count()` are not acceptable. Another example is subscribing to a proxy state. The library exports a function called `subscribe` and can be used like this:

```js
subscribe(state, () => {
  console.log(state.count);
});
```

We can't use the method on the state object like this:

```js
state.subscribe(() => {
  console.log(state.count);
});
```

This is because it can create name collisions. We could have used a symbol like this:

```js
state[SUBSCRIBE](() => {
  console.log(state.count);
});
```

However, the exported function is more explicit. In fact, we use a symbol internally.

Another benefit of the exported `subscribe` function is that it's minifiable. The Valtio API and implementation are designed to be as minifiable as possible.

### The Secret Sauce in Snapshot Creation

The idea of a snapshot isn't very difficult, but the implementation covers various scenarios, including:

- Nested objects, obviously
- Lazy evaluation
- Caching to avoid re-evaluation
- Circular reference support

If you remember ImmutableJS, the implementation design is likely similar.

As mentioned earlier, the snapshot idea is important not only for the React contract but also for the `proxy-compare` library. Internally, Valtio consists of two parts: `valtio/vanilla` to create proxy state and snapshots, and `valtio/react` to provide the hooks API with `proxy-compare`.

If you are interested, please check out my blog posts:

- [How Valtio Proxy State Works (Vanilla Part)](https://blog.axlight.com/posts/how-valtio-proxy-state-works-vanilla-part/)
- [How Valtio Proxy State Works (React Part)](https://blog.axlight.com/posts/how-valtio-proxy-state-works-react-part/)

### Current Status

Valtio has been around for a while and is widely used in production. The current version, `v1`, is quite stable, but we will soon release `v2`. It's meant to address some feedback from v1 that requires breaking changes and to ensure compatibility with React 19, which introduces the new `use` hook.
