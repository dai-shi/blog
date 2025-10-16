---
title: "Why Zustand Typescript Implementation Is So Ugly"
subtitle: "BTW, JavaScript implementation is so clean"
date: 2023-07-19T15:00:00+09:00
tags: ["react", "global-state", "typescript", "zustand"]
---

### Introduction

Note: This post focuses on Zustand library's implementation in TypeScript.
It does not affect the user code, which should be kept clean.

Zustand's JavaScript implementation is very small, as seen in the following tweet.

{{< x user="dai_shi" id="1583082766081531905" >}}

However, its TypeScript implementation is quite complicated.
There are several reasons, one of which we explore in this post.

### SetStateInternal type

Let's take a closer look at the `SetStateInternal` type.

(Keep in mind that it's intended for internal use, as we recommend using `StoreApi['setState']` for public APIs.)

https://github.com/pmndrs/zustand/blob/f11cc7f37ad0a53f49370c0e4e659690a1c1a721/src/vanilla.ts#L1-L6

```ts
type SetStateInternal<T> = {
  _(
    partial: T | Partial<T> | { _(state: T): T | Partial<T> }['_'],
    replace?: boolean | undefined
  ): void
}['_']
```

What are those simily characters `['_']`?

It's a TypeScript hack, and the actual character used (in this case, `_`)
is not significant; any word would work.
We don't go into TypeScript technical details,
but let's see what happens if we don't use such a hack.

What we would imagine is the following type as a simpler alternative.

```ts
type SetStateInternal2<T> = (
  partial: T | Partial<T> | ((state: T) => T | Partial<T>),
  replace?: boolean | undefined
) => void
```

This doesn't work for certain cases.
Let's explore how it fails with contrived examples.

The first one is the following:

```ts
type State = { name: string; age: number };

declare function run<Fn extends SetStateInternal<unknown>>(fn: Fn): void;
declare function run2<Fn extends SetStateInternal2<unknown>>(fn: Fn): void;
declare const setState: SetStateInternal<State>
declare const setState2: SetStateInternal2<State>
run(setState);
run2(setState2);
```

`run2` causes a type error.

The second one is the following:

```ts
type NameOnlyState = { name: string };
type StoreApi<T> = { setState: SetStateInternal<T> };
type StoreApi2<T> = { setState: SetStateInternal2<T> };
declare const store: StoreApi<State>;
declare const store2: StoreApi2<State>;
const nameOnlyStore: StoreApi<NameOnlyState> = store;
const nameOnlyStore2: StoreApi2<NameOnlyState> = store2;
```

Assigning to `nameOnlyStore2` fails.

You can find the full reproduction in the TypeScript playground: https://tsplay.dev/NBVO4N

As a library consumer, you don't need to know why this happens or how to resolve it.
It's something TypeScript magicians should take care of.

### Closing notes

This post has highlighted one of the reasons
behind the complexity of Zustand's TypeScript implementation.
There are other reasons, which could be addressed in a future post.
