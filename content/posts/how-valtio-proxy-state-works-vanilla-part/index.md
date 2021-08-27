---
title: "How Valtio Proxy State Works (Vanilla Part)"
subtitle: "Adding immutability to mutable state"
date: 2021-08-27T22:00:00+09:00
tags: ["global-state", "proxy"]
---

## Introduction

[Valtio](http://github.com/pmndrs/valtio) is a library for
global state primarily for React.
It's originally modeled to match with
[useMutableSource](https://github.com/reactjs/rfcs/blob/master/text/0147-use-mutable-source.md)
API.
However, it turns out it's a novel API to add
immutability to mutable state.

What is immutable state? JavaScript doesn't support
immutability as language, so it's just a coding contract.

```js
const immutableState1 = { count: 0, text: 'hello' };

// update the state
const immutableState2 = { ...immutableState1, count: immutableState1.count + 1 };

// update it again
const immutableState3 = { ...immutableState2, count: immutableState2.count + 1 };
```

Some people might be familiar with this pattern,
or it can be new to some other people.
We always create a new object without modifying the existing ones.
This allows us to compare the state objects.

```js
immutableState1 === immutableState2 // is false
immutableState2 === immutableState3 // is false

// decrement count
const immutableState4 = { ...immutableState3, count: immutableState3.count - 1 };

console.log(immutableState4); // shows "{ count: 1, text: 'hello' }"
console.log(immutableState2); // shows "{ count: 1, text: 'hello' }"

// however their references are different
immutableState2 === immutableState4 // is false
```

The benefit of immutable states is you can compare the state object
with `===` to know if anything inside can be changed.

Contradictory to immutable state, mutable states are JavaScript objects
without any contracts on updating.

```js
const mutableState = { count: 0, text: 'hello' };

// update the state
mutableState.count += 1;

// update it again
mutableState.count += 1;
```

Unlike immutable state, we mutate state and keep the same object.
Because it's how JavaScript objects are mutable by nature,
mutable state is easier to handle.
The problem of mutable states is the flip side of the benefit
of immutable states.
If you have two mutable state objects,
you need to compare all properties to see if they have same contents.

```js
const mutableState1 = { count: 0, text: 'hello' };
const mutableState2 = { count: 0, text: 'hello' };

const isSame = Object.keys(mutableState1).every(
  (key) => mutableState1[key] === mutableState2[key]
);
```

This is not enough for nested objects and also the number of keys
can be different.
You need so-called deepEqual to compare two mutable objects.

deepEqual is not very efficient for large objects.
Immutable objects shine there because the comparison
doesn't depend on the size nor the depth of objects.

So, our goal is to bridge between mutable state and immutable state.
More precisely, we want to automatically create
immutable state from mutable state.

## Detecting mutation

Proxy is a way to trap object operations.
We use `set` handler to detect mutations.

```js
const p = new Proxy({}, {
  set(target, prop, value) {
    console.log('setting', prop, value);
    target[prop] = value;
  },
});

p.a = 1; // shows "setting a 1"
```

We need to track if the object is mutated,
so it has a version number.

```js
let version = 0;
const p = new Proxy({}, {
  set(target, prop, value) {
    ++version;
    target[prop] = value;
  },
});

p.a = 10;
console.log(version); // ---> 1
++p.a;
console.log(version); // ---> 2
```

This version number is for the object itself,
and it doesn't care which property is changed.

```js
// continued
++p.a;
console.log(version); // ---> 3
p.b = 20;
console.log(version); // ---> 4
```

As we can now track the mutation,
next is to create an immutable state.

## Creating snapshot

We call an immutable state of a mutable state, a snapshot.
We create a new snapshot if we detect mutation,
that is when the version number is changed.

Creating a snapshot is basically copying an object.
For simplicity, let's assume our object is not nested.

```js
let version = 0;
let lastVersion;
let lastSnapshot;
const p = new Proxy({}, {
  set(target, prop, value) {
    ++version;
    target[prop] = value;
  },
});
const snapshot = () => {
  if (lastVersion !== version) {
    lastVersion = version;
    lastSnapshot = { ...p };
  }
  return lastSnapshot;
};

p.a = 10;
console.log(snapshot()); // ---> { a: 10 }
p.b = 20;
console.log(snapshot()); // ---> { a: 10, b: 20 }
++p.a;
++p.b;
console.log(snapshot()); // ---> { a: 11, b: 21 }
```

`snapshot` is a function to create a snapshot object.
It's important to note that the snapshot object is
only created when `snapshot` is invoked.
Until then, we can do as many mutations as we want,
which only increment `version`.

## Subscribing

At this point, we don't know when mutations happen.
It's often the case we want to do something if the state is changed.
For this, we have subscriptions.

```js
let version = 0;
const listeners = new Set();
const p = new Proxy({}, {
  set(target, prop, value) {
    ++version;
    target[prop] = value;
    listeners.forEach((listener) => listener());
  },
});
const subscribe = (callback) => {
  listeners.add(callback);
  const unsubscribe = () => listeners.delete(callback);
  return unsubscribe;
};

subscribe(() => {
  console.log('mutated!');
});

p.a = 10; // shows "mutated!"
++p.a; // shows "mutated!"
p.b = 20; // shows "mutated!"
```  

Combining `snapshot` and `subscribe` allows us
to connect mutable state to React.

We will introduce how valtio works with React
in another post.

## Handling nested objects

So far, our examples were with simple objects,
whose properties are primitive values.
In reality, we want to use nested objects,
and it's the benefit of immutable state.

Nested object looks something like this.

```js
const obj = {
  a: { b: 1 },
  c: { d: { e: 2 } },
};
```

We would also like to use arrays.

Valtio supports nested objects and arrays.
If you are interested in how it's implemented,
check out the source code.

<https://github.com/pmndrs/valtio>

## Closing notes

In this blog post, we use simple code in examples.
The implementation does something more
to handle various cases. It's still bare minimum.

The actual API is very close to the example code.
Here's rough type definition in TypeScript.

```ts
function proxy<T>(initialObject: T): T;

function snapshot<T>(proxyObject: T): T;

function subscribe<T>(proxyObject: T, callback: () => void): () => void;
```

In this post, we discussed about the vanilla part of valtio.
Hope to write about the react part, some time soon.
