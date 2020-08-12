---
title: "Developing React Global State Library With Atom Abstraction"
date: 2020-08-13T08:00:00+09:00
tags: ["react", "hooks", "global-state", "atom", "recoil", "concurrent"]
---

## Introduction

I have been developing various global state libraries for React.
For example:
- [react-tracked](https://github.com/dai-shi/react-tracked)
- [react-hooks-global-state](https://github.com/dai-shi/react-hooks-global-state)

My main motivation is to eliminate selector functions that
are only required for render optimization.
Render optimization here means it avoids extra re-renders.
An extra re-render is a re-render process that produces
the same view result as before.

Since [Recoil](https://github.com/facebookexperimental/Recoil) is announced,
I'm very interested in atom abstraction because it eliminates
selector functions for render optimization and the API seems pretty intuitive.

I couldn't help stopping creating something by myself.
This post introduces my challenges up to now with some notes.

## Recoildux

My first challenge was to use a Redux store as an atom.
Redux itself is very lightweight. Although the ecosystem
assumes only one Redux store exists in an app,
we could technically create as many stores as we want.

I have already developed a react redux binding for Concurrent Mode.
It uses the upcoming
[useMutableSource](https://github.com/reactjs/rfcs/pull/147)
hook, and most notably it doesn't depend on React Context.

[reactive-react-redux](https://github.com/dai-shi/reactive-react-redux) is
the repository and especially [#48](https://github.com/dai-shi/reactive-react-redux/pull/48) has the code as of writing.

Based on it, the implementation is pretty straightforward.
Only the challenge is how to create a new atom based on existing atoms.
I wanted to something similar to `combineReducers`,
and created `combineAtoms`. Unfortunately, it didn't go well:
a) the API is not very flexible and b) the implementation is too hacky.

On the other hand, Recoil-like `selector` is implemented more cleanly
and it's flexible.

Here's the repository.

<https://github.com/dai-shi/recoildux>

Unfortunately, the current implementation of atom has builtin reducer,
and we can't use custom reducer. Most of Redux ecosystem is not very
usable for this, so it doesn't get much benefit of using Redux.
(Except that I could build it fairly quickly based on reactive-react-redux v5-alpha.)

## react-hooks-global-state

I have been developing this for a long time with various assumptions.

<https://github.com/dai-shi/react-hooks-global-state>

As of writing, v1 is pretty stable based on subscription model.
v2-alpha is also implemented with useMutableSource and should be
Concurrent Mode compatible.

The current API is mainly for a single store or a small set of them.
Based on my first challenge with Recoildux, I was pretty sure
atom abstraction is possible and easy without derived atoms.
Still, there are a few benefits. a) It allows the pattern
of small and many stores. b) It enables code splitting.

v1 compatible APIs are simple wrappers around atom abstraction.
So, even without derived atoms (= Recoil's `selector`),
atom abstraction makes a certain sense.

Here's the code as of writing.

<https://github.com/dai-shi/react-hooks-global-state/pull/38>

I'd say the implementation has nothing special.
It's only about the use of the terminology "atom" which means small store in this case.

## use-atom

The previous two libraries are for so called external stores.
It means stores are created outside of React.
That is totally fine. However, in Concurrent Mode it's
recommended to use React state for state branching.
For more information about Concurrent Mode,
check out [the React doc](https://reactjs.org/docs/concurrent-mode-intro.html).

I have been developing [react-tracked](https://github.com/dai-shi/react-tracked)
and know how difficult to create global state only with React state.

Luckily, there is a library to ease it,
which is [use-context-selector](https://github.com/dai-shi/use-context-selector).
Based on this, it would only require small effort to create a new library with atom abstraction.

Here's the repository.

<https://github.com/dai-shi/use-atom>

Contradictory to my expectation, it was extremely hard to implement.
There are many reasons but some notable ones are:

1. API seems simple and intuitive, but that doesn't mean the implementation is simple. For example, it's hard to know if an updating action is sync or async. We'd want to show loading indicator only if the action is async.
2. Handling atom dependencies is not trivial. We need to create a dependency graph, but we don't know it in advance. We can only know it at runtime. Furthermore, there's no way to remove dependencies if they are no longer dependent.
3. It's almost impossible to suspend correctly for React Suspense. It's already noted above, but we can't know what is async and what is dependent.

The current version of use-atom tries to do its best.
There's various edge cases that it doesn't work as expected.
I don't think the implementation is polished, and
there might be better way we would find in the future.

Note that use-atom has a compatibility layer with Recoil.
But, it doesn't fully replicate the API and there are some limitations
and inconsistencies. Nevertheless, it's compatible for simple cases,
and we can compare the behavior between use-atom and Recoil.

## Closing notes

It was nice experience trying these challenges.
One of big findings for me is simple API for users
is not always easy to implement. This may sounds obvious,
but it's something I learnt from this.
If the implementation is hard, it's likely to have more bugs.
I hope to find a variant of atom abstraction
that is intuitive to users and not complicated to implement.
