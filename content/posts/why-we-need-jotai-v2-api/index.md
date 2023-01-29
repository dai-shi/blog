---
title: "Why We Need Jotai v2 API"
subtitle: "It will be incompatible with Recoil"
date: 2022-12-06T14:00:00+09:00
tags: ["react", "hooks", "global-state", "atom", "jotai"]
---

## Introduction

[Jotai](https://github.com/pmndrs/jotai)
is a library for React state management.

The API (let's call it v1 API) is designed to
a) be friendly with Concurrent React, and
b) be compatible with Recoil as much as possible.

What does it mean?
First, atom read function is evaluated in the render phase in React.

For example, consider a simple derived atom.

```js
const countAtom = atom(0);
const doubleAtom = atom((get) => get(countAtom) * 2);
```

In the example, `(get) => get(countAtom) * 2` is the read function.
When `countAtom` is updated, the library doesn't
invoke the read function of `doubleAtom` immediately.
Instead, it triggers the component that uses `doubleAtom` to re-render.
The component re-renders eventually, and
it's when the read function of `doubleAtom` is invoked.
The benefit of this behavior might be hypothetical,
but React allows to schedule re-renders, which means
it can skip invoking read functions in concurrent rendering.

Another behavior in v1 API is that
async atoms can suspend and replay.

For example, let's see an example.

```js
const countAtom = atom(0);
const asyncAtom = atom(async (get) => {
  const isEven = get(countAtom) % 2 === 0;
  await someAsyncFunction();
  return isEven;
});
const derivedAtom = atom((get) => {
  const isEven = get(asyncAtom);
  return isEven ? "even" : "odd";
});
```

It might be a little complex. The first atom is a primitive atom,
the second atom is an async atom that depends on the first atom,
and the third atom is a "sync" atom that depends on the
second atom which is "async".

When `countAtom` is changed and the read function of `derivedAtom` is
invoked, then the read function of `asyncAtom` is evaluated and,
it suspends because it returns a promise.
When the promise is resolved, `asyncAtom` has a resolved value,
and the Jotai library re-evaluates the read function of `derivedAtom`.

The read function of `derivedAtom` may return the same value when `countAtom`
value is increased by say two.
In such a case, the rendering bails out and it doesn't make changes to DOM.

That is basically how v1 API is designed.
It's been a while since the initial release, 
and it turns out that v1 API has some limitations.

## Two major problems

Evaluating the read function is intentional as described,
but the behavior is a little difficult to understand.

Consider the following example:

```js
const countAtom = atom(0);
const derivedAtom = atom((get) => {
  const isEven = get(countAtom);
  return isEven ? "even" : "odd";
});
```

When the `countAtom` value is increased by two,
the read function of `derivedAtom` is re-evaluated,
but it returns the same value as before.
In React 17, it can't be observed because of early bail out in `useReducer`.

In React 18, the `useReducer` behavior is changed and it does't early bail out.
It's a good thing for the Jotai library, because we want to
delay the evaluation of read functions.
However, this behavior is very confusing to many developers,
because people often use `console.log` to debug rendering behaviors.

See also an example in [this tweet](https://twitter.com/dai_shi/status/1534170089981100032).

We call this behavior "extra re-renders without commits",
and it's an expected behavior as a design.
If this behavior can cause heavy computation, we can use `useMemo` to avoid it.
However, it's confusing anyways.

Another issue is with store API.
We have an internal store API in v1 API,
and we try to expose it for more use cases with `unstable_createStore`.
Its difficulty is with async atoms.
We can't expose "suspend and replay" behavior,
because it's not very understandable.

```js
const store = unstable_createStore();
const countAtom = atom(0);
const asyncAtom = atom(async (get) => {
  await new Promise((r) => setTimeout(r, 1000));
  return get(countAtom);
});

store.get(asyncAtom); // returns `undefined` while asyncAtom is pending.
store.getAsync(countAtom); // returns a promise even though countAtom is not async.
```

It's hypothetically impossible to provide a better API,
because we can't distinguish if an atom will suspend or not.
(A sync atom that depends on an async atom may suspend.)

Meanwhile, the React team opens a new RFC with a new hook named `use`.

## The `use` RFC

The RFC is here: https://github.com/reactjs/rfcs/pull/229

As of writing, that RFC is not yet finalized,
but it's enough to make us re-think the Jotai API.
The new `use` hook can resolve a promise with the suspend-and-replay behavior,
just like we could throw a promise previously.
However, it doesn't allow to use `use` anywhere.
You can only use `use` in other hooks.
It means that the suspend-and-replay behavior that's
done in Jotai library can't be replaced with `use`.

Furthermore, the RFC states we should prefer async/await if possible,
and the `use` hook is an unavoidable solution for now.

This made us to consider new Async API.

## New Async API

We already distinguish sync atoms and async atoms.

```js
const syncAtom = atom(() => 0); // Atom<number>
const asyncAtom = atom(async () => 0); // Atom<Promise<number>>
```

However, when we use `get` function to get atom values,
it returns same values.

```js
const derived1 = atom((get) => get(syncAtom)); // Atom<number>
const derived2 = atom((get) => get(asyncAtom)); // Atom<number>
```

This is possible because it suspends and replays with async atoms.
It means the `derived2` can suspend because it depends on
an async atom even though the signature of `derive2` is a sync atom.

The new API avoids such behavior and just expose promises.
It doesn't do anything special to handle promises.
The `get` function returns a value even if it's a promise,
instead of resolving the promise.

```js
// New API
const derived1new = atom((get) => get(syncAtom)); // Atom<number>
const derived2new = atom((get) => get(asyncAtom)); // Atom<Promise<number>>
```

With the new API, it's explicit what atoms will suspend from their types.
Only atoms with promise values can suspend.

What's the downside of the new API?
We can no longer read async atom values in sync, obviously.

To use async atom values in sync, `loadable` util may help to certain extent.

```js
// New API
const derived3new = atom((get) => get(loadable(asyncAtom))); // Atom<Loadable<number>>>
```

We also consider adding a new util called `unwrap` (tentative name).

```js
// New API
const derived4new = atom((get) => get(unwrap(asyncAtom, -1))); // Atom<number>>
```

Note that `-1` is a default value which is used while `asyncAtom` doesn't have a value initially.

We will release the new Async API in v2.
While the v1 API is heavily inspired by Recoil,
the v2 API is not inspired by Recoil,
and is incompatible with Recoil API.
(The sync atom behavior is probably compatible.)

We will learn how the v2 API goes.
There might be some difficulties.
Having that said, it seems promising at the moment,
because the async behavior is rather straightforward
without "suspend and replay".
It makes it possible to expose the store API,
which is a long awaited feature for some Jotai users.

The biggest problem is that the new Async API is breaking change.
Most importantly, the syntax isn't changed, and only behavior is changed.
Hopefully, most Jotai users would use TypeScript, and in such case,
it's a little easier to notice the change because TypeScript
complier complains the invalid usage.

## Migration strategy

It would have been easier if the change were in syntax,
and we could have both new one and old one and deprecate the old one.

The thing is it's only breaking for async atoms.
We assume quite a few users use Jotai only with sync atoms,
and there's nothing breaking with sync atoms.

We need a major version up for the new API,
thus it's going to be v2 API.

To mitigate migrating to the new API,
we will provide it in different entry points.

```js
// New API
import { atom } from 'jotai/vanilla';
import { useAtom } from 'jotai/react';
```

This allows us to pre-release the new API in v1.

There's another reason why we provide it in different entry points.
It would allow using Jotai in non-React environments.
For example, there's a project to run Jotai apps without React.
See [a PR of the project](https://github.com/jotaijs/jotai-jsx/pull/5).

We considered if we should deprecate v1 API before jumping to v2,
but it would just be a trouble for sync atom users.
Some feedbacks suggested to directly jumping to v2.

So, our plan is to pre-release the new API in v1,
and later release v2 to have only the new API.

```js
// In v2
import {
  atom, // same as import { atom } from 'jotai/vanilla';
  useAtom, // same as import { useAtom } from 'jotai/react';
} from 'jotai';
```

That way allows for sync atom users to simply update to v2
without any problems.

It also allows async atom users to try the new API in v1 and migrate to it,
and the migrated code will work in v2 too.

## Closing note

We already tried the new API with various integration libraries as follows:

- https://github.com/jotaijs/jotai-location/pull/1
- https://github.com/jotaijs/jotai-form/pull/12
- https://github.com/jotaijs/jotai-tanstack-query/pull/18
- https://github.com/jotaijs/jotai-urql/pull/5
- https://github.com/jotaijs/jotai-relay/pull/3
- https://github.com/jotaijs/jotai-optics/pull/4
- https://github.com/jotaijs/jotai-immer/pull/1
- https://github.com/jotaijs/jotai-trpc/pull/8
- https://github.com/jotaijs/jotai-xstate/pull/1
- https://github.com/jotaijs/jotai-redux/pull/1
- https://github.com/jotaijs/jotai-zustand/pull/2
- https://github.com/jotaijs/jotai-valtio/pull/1
- https://github.com/jotaijs/jotai-jsx/pull/5
- https://github.com/jotaijs/jotai-game/pull/1
- https://github.com/jotaijs/jotai-suspense/pull/2
- https://github.com/jotaijs/jotai-cache/pull/1
- https://github.com/dai-shi/use-atom/pull/33

We encourage all Jotai users, whether they use async atoms or sync only,
to try the new Async API (a.k.a. v2 API) pre-released since
[jotai v1.11.0](https://github.com/pmndrs/jotai/releases/tag/v1.11.0).
