---
title: "How Jotai Was Born"
subtitle: "Is it just an alternative to Recoil?"
date: 2024-08-09T22:00:00+09:00
tags: ["react", "global-state", "typescript", "jotai"]
---

### Introduction

In this post, I would like to share the story of why I started developing Jotai. While Jotai is often seen as a similar solution to Recoil, there is a longer history behind its development.

### React Hooks

It was October 2018 when React Hooks were first announced. I liked the idea of developing logic outside of React components, and believed that many libraries would soon adopt this approach. I wanted to develop something, and global state management was one field I picked. My motivation was to avoid Redux selectors, also known as `mapStateToProps` at that time. In my work experience, I found that it's pretty hard for beginners to understand referential equality. To avoid extra re-renders, we may need to use memoization libraries such as `reselect`. It felt unnecessary if the computation of selectors themselves was fairly lightweight.

### Reactive React Redux

My initial goal was to develop a binding library using React Hooks for React Redux that avoided the need for selectors. To be clear, selectors themselves are fine. What I wanted to avoid was using selectors only for render optimization. What I reached for was Proxies. Proxies allow us to track the access of state objects, and we can only trigger re-renders if the accessed properties are changed. This eliminated the need for selectors for render optimization. It automatically optimized re-renders. Some people liked the idea, but meanwhile, the official React Redux binding provided a `useSelector` hook and it became the de-facto standard. Eventually, my library was marked as unmaintained.

<https://github.com/dai-shi/reactive-react-redux>

### React Tracked

With Redux having its own Hooks library, I decided to move away from Redux and develop a library that focused on the React Context pattern. A naive solution with `useState` (or `useReducer`) with React Context has the extra re-render issue. This library took the same approach as the previous one and overcame the issue. The biggest benefit of this library is that it doesn't change the semantics from the naive solution. If people had already used `useState` and propagated the state through the context, introducing the library is very easy and renders are optimized automatically. This library is called React Tracked.

<https://github.com/dai-shi/react-tracked>

I put much effort into this library, did benchmarks, wrote documentation, set up the website, and wrote a bunch of blog posts. Some people really liked it and it was used in production. However, something was missing to get more attention. The negative feedback was that it worked too magically behind the scenes, and some people hesitated to use Proxies.

### Recoil

I had been exploring ways to avoid using Proxies while maintaining the goal of no selectors for render optimization. In May 2020, Recoil was presented at the React Europe conference. That was exactly what I wanted. The reason I made React Tracked with React Context was to make it compatible with Concurrent Mode. Recoil was said to be compatible with Concurrent Mode. The atom model was slightly different from observables, and focused on the React use case.

<https://www.youtube.com/watch?v=_ISAA_Jt9kI>

I was kind of shocked that I didn't come up with similar ideas. The fundamental technique wasn't very new. MobX did something pretty similar. As Recoil was announced, I half gave up creating my own library. I thought we could just use it. But that didn't stop me from thinking about it and experimenting with something similar. One thing I didn't like about Recoil was that an atom required a string key. The API didn't feel right. While it's understandable that it's for serialization, that actually convinced me that Recoil would never allow omitting string keys. So, I decided to develop my own library.

### Jotai

Jotai was obviously inspired by Recoil, but there are several key differences. Beyond omitting string keys, its design philosophy focuses on a small API surface and a minimal core. Any additional features should be implemented outside the core. Jotai was first announced on September 9, 2020. At that point, it was very uncertain how people would consider it. I thought Recoil was like Flux and there would be many Recoil-like libraries. The goal was to promote the idea of the atom model.

<https://github.com/pmndrs/jotai>

After the first announcement, there was some positive feedback and I continued to develop it, reached v1, and then reached v2. There were more contributors coming and we now maintain the project as a group. We have docs and a website. We also have many integration libraries around the minimal core, maintained by several contributors.

### Current Status

We've addressed various bugs, and Jotai is now quite stable. My current focus is on making the core more maintainable by contributors. The core code is very complex, and my challenge is to make it more readable and understandable. More integration libraries would be nice and they are increasing, actually.

Thanks to all Jotai users and contributors. I'm very grateful for it.
