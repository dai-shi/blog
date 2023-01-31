---
title: "You Might Not Need React Query for Jotai"
subtitle: "Explore data fetching solution with atoms"
date: 2023-01-31T14:00:00+09:00
tags: ["react", "hooks", "global-state", "atom", "jotai", "fetch"]
---

## Introduction

When Jotai development was started (before releasing v1),
it has a simple goal. It optimizes re-renders, which was
often a problem with useState and useContext using a big state object.
We also wanted to avoid using selector function,
which is popularized by Redux and widely used.

In the early days, I wanted to have data fetching solution,
but didn't want to complicate Jotai itself.
So, `jotai/query` package was created.
It's an integration with React Query v3.
It went well, but I noticed a certain mismatch with users.
While my mental model was to create a data fetching API _for_ Jotai,
people want full features from React Query.
I had been struggling with the mismatch, and
ended up with creating a new library `jotai-tanstack-query`.
It integrates TanStack Query v4 and covers full features.

So, now I'm back to the original challenge.
What would be a data fetching API for Jotai?
This post is to explore that road.
We may not reach a conclusion that works right away, but let's start.

## What Jotai can do

Jotai has a store internally, which holds values of atoms.
It's implemented with `WeakMap` whose key is an atom itself,
and its value of an atom is the latest value.
The use of `WeakMap` allows to avoid memory leaks.
The store has only one value per atom.
It doesn't keep the history of changes.

In Jotai v2 API, `createStore` is exposed and
we can directly read and write atom values.

```js
import { createStore, atom } from 'jotai/vanilla';

const store = createStore();

const atom1 = atom();
const atom2 = atom();

store.set(atom1, 'hello');
store.set(atom2, 'world');

console.log(store.get(atom1)); // ---> 'hello'
```

We could use the store for caching.
Atoms can be defined with async functions.
Here's a simple data fetching example.

```js
const idAtom = atom(1);
const dataAtom = atom(async (get) => {
  const id = get(idAtom);
  const res = await fetch(`https://reqres.in/api/posts/${id}`);
  const data = await res.json();
  return data;
});
```

`dataAtom` depends on `idAtom` and it will be only
re-evaluated when a dependency changes.
So, it works like a very basic cache capability.

The strength is that we can derived another atom from `dataAtom`.
It allows to form so-called data flow graph.

We can also use some utils like `atomWithDefault` so that
we can override the cache value.

The point is, once `dataAtom` is defined,
we could do everything with full Jotai functionalities.
From the library perspective, it doesn't know if
`dataAtom` is a cache data from somewhere else.

## What Jotai can't do

It seems like we can do very simple data fetching only with Jotai.
So, what's missing?
Well, there are so many things, but let's focus on something.
My motivation for the original `jotai/query` is richer caching.

Because Jotai store can only hold current value, it's very limited for caching.
Let's revisit the previous example.

```js
const idAtom = atom(1);
const dataAtom = atom(async (get) => {
  const id = get(idAtom);
  const res = await fetch(`https://reqres.in/api/posts/${id}`);
  const data = await res.json();
  return data;
});
```

If we change the value of `idAtom` from `1` to `2`,
the async function will be invoked and the value of `dataAtom` will be updated.
What if we change it back from `2` to `1`?
Because Jotai store holds only the latest value of `dataAtom`,
it will re-invoke the async function with id=1, which it did previously.
This is not ideal, because our async function is fetching data from
a server, and data fetching is usually costly.

A side note: The previous data can actually be cached in browser or somewhere. However, having async value (= promise) is not nice in React, because it will trigger render twice. The newly proposed `use` hook might mitigate this issue.

Having only the latest values s one of huge limitations
with Jotai store, if we use it for cache for data fetching.
But, it's by design. How can we overcome such a limitation?

## jotai-cache package

Jotai's API is very minimal and it encourage adding features outside.
Atoms are composable and we can implement various features.
There are many libraries to provide various features.

jotai-cache is one of such libraries that tries to add
richer caching feature for Jotai atoms.

<https://github.com/jotaijs/jotai-cache>

As of writing, it has `atomWithCache` function.
It solves the caching problem described in the previous section.
Its basic usage is straightforward. Just replace `atom` with `atomWithCache`.

```js
import { atomWithCache } from 'jotai-cache';

const idAtom = atom(1);
const dataAtom = atomWithCache(async (get) => {
  const id = get(idAtom);
  const res = await fetch(`https://reqres.in/api/posts/${id}`);
  const data = await res.json();
  return data;
});
```

Now, if we change the value of `idAtom` from `2` back to `1`,
it won't re-invoke the async function, but will use cached value.

This is just one capability for data fetching and it's the start.
We hope to have more features in this lib.

`atomWithCache` is something I wanted when I created `jotai/query`.
We would like to explore Jotai oriented solutions for data fetching.

## Closing notes

Well, it's still something in progress.
From Jotai point of view, atoms are building blocks,
and we should be able to create data fetching solution.
That doesn't mean it can reach the level of TanStack Query.
They have more features.
For examples, infinite query, mutation, and optimistic updates
are probably tough ones.
There are probably lots to learn from existing solutions.
We wouldn't need to implement all the features.
If our app is 99% with Jotai and want to add data featching solution,
a small missing capability would be nice to fulfill our requirements.
On the other hand,
you might not need Jotai if TanStack Query covers your needs.
