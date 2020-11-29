---
title: "Developing a Memoization Library With Proxies"
subtitle: "proxy-compare and proxy-memoize"
date: 2020-11-29T14:00:00+09:00
tags: ["proxy", "memoization", "react", "redux"]
---

### Introduction

It's been a while since I started developing
[reactive-react-redux](https://github.com/dai-shi/reactive-react-redux)
and [react-tracked](https://github.com/dai-shi/react-tracked).
These libraries provide so called
[state usage tracking](https://blog.axlight.com/posts/what-is-state-usage-tracking-a-novel-approach-to-intuitive-and-performant-api-with-react-hooks-and-proxy/) to optimize render in React.
This approach, I think, is pretty novel and
quite a lot of my effort has been put to improve its performance.

Lately, I thought it would be nicer if this can be used more broadly.
I wondered if it can be used in vanilla JS.
What would be an API in vanilla JS?
It would be good if it's easy to understand.
My idea ended up with memoization, mainly because
the primary goal is to be a replacement of
[reselect](https://github.com/reduxjs/reselect).

The new library is named `proxy-memoize`.

### proxy-memoize

GitHub: <https://github.com/dai-shi/proxy-memoize>

The `proxy-memoize` library provides a memoize function.
It will take a function and return a memoized function.

```js
import memoize from 'proxy-memoize';

const fn = (x) => ({ foo: x.foo });
const memoizedFn = memoize(fn);
```

There's a big design choice in this library.
A function to be memoized must be a function
which takes exactly one object as an argument.
So, functions like below are not supported.

```js
const unsupportedFn1 = (number) => number * 2;

const unsupportedFn2 = (obj1, obj2) => [obj1.foo, obj2.foo];
```

This will allow caching the results with `WeakMap`.
We can cache as many results as we want and let JS garbage collect
when they are no longer effective.

Proxies are used if we don't find a result in the `WeakMap` cache.
The memoized function invokes the original function
with the argument object wrapped by proxies.
The proxies track the usage of object properties while invoking the function.
The tracked information is called "affected," which
is a partial tree structure of the original object.
For simplicity, we use dot notation in this post.

Let's look at following examples.

```js
const obj = { a: 1, b: { c: 2, d: 3 } };

// initially affected is empty

console.log(obj.a) // touch "a" property

// affected becomes "a"

console.log(obj.b.c) // touch "b.c" property

// affected becomes "a", "b.c"
```

Once "affected" is created, it can check a new object
if the affected properties are changed.
Only if any of affected properties is changed,
it will re-invoke the function.
This will allow very fine tuned memoization.

Let's see an example.

```js
const fn = (obj) => obj.arr.map((x) => x.num);
const memoizedFn = memoize(fn);

const result1 = memoizedFn({
  arr: [
    { num: 1, text: 'hello' },
    { num: 2, text: 'world' },
  ],
})

// affected is "arr[0].num", "arr[1].num" and "arr.length"

const result2 = memoizedFn({
  arr: [
    { num: 1, text: 'hello' },
    { num: 2, text: 'proxy' },
  ],
  extraProp: [1, 2, 3],
})

// affected properties are not change, hence:
result1 === result2 // is true
```

The usage tracking and affected comparison is
done by an internal library "proxy-compare."

### proxy-compare

GitHub: <https://github.com/dai-shi/proxy-compare>

This is a library that is extracted from react-tracked
to only provide a comparison feature with proxies.
(Actually, react-tracked v2 will use this library as a dependency.)

The library exports two main functions: `createDeepProxy` and `isDeepChanged`

It works like the following:

```js
const state = { a: 1, b: 2 };
const affected = new WeakMap();
const proxy = createDeepProxy(state, affected);
proxy.a // touch a property
isDeepChanged(state, { a: 1, b: 22 }, affected) // is false
isDeepChanged(state, { a: 11, b: 2 }, affected) // is true
```

The `state` can be a nested object, and only when
a property is touched, a new proxy is created.
It's important to note `affected` is provided from outside,
which will ease integrating this in React hooks.

There are other points about performance improvements
and dealing with edge cases.
We don't go too much in detail in this post.

### Usage with React Context

As discussed in [a past post](https://blog.axlight.com/posts/4-options-to-prevent-extra-rerenders-with-react-context/),
one option is to use useMemo.
If proxy-memoize is used with useMemo, we would be
able to get a similar benefit like react-tracked.

```jsx
import memoize from 'proxy-memoize';

const MyContext = createContext();

const Component = () => {
  const [state, dispatch] = useContext(MyContext);
  const render = useMemo(() => memoize(({ firstName, lastName }) => (
    <div>
      First Name: {firstName}
      <input
        value={firstName}
        onChange={(event) => {
          dispatch({ type: 'setFirstName', firstName: event.target.value });
        }}
      (Last Name: {lastName})
      />
    </div>
  )), [dispatch]);
  return render(state);
};

const App = ({ children }) => (
  <MyContext.Provider value={useReducer(reducer, initialState)}>
    {children}
  </MyContext.Provider>
);
```

The `Component` will re-render when context changes. However,
it returns memoized react element tree unless `firstName` isn't changed.
So, re-render stops there. This behavior is different from
react-tracked, but it should be fairly optimized.

### Usage with React Redux

It can be a simple replacement to reselect.

```jsx
import { useDispatch, useSelector } from 'react-redux';
import memoize from 'proxy-memoize';

const Component = ({ id }) => {
  const dispatch = useDispatch();
  const selector = useMemo(() => memoize((state) => ({
    firstName: state.users[id].firstName,
    lastName: state.users[id].lastName,
  })), [id]);
  const { firstName, lastName } = useSelector(selector);
  return (
    <div>
      First Name: {firstName}
      <input
        value={firstName}
        onChange={(event) => {
          dispatch({ type: 'setFirstName', firstName: event.target.value });
        }}
      />
      (Last Name: {lastName})
    </div>
  );
};
```

This might be too simple to show the power of proxy-memoize,
one of interesting use cases would be the following.

```jsx
memoize((state) => state.users.map((user) => user.firstName))
```

This will only be re-evaluated if the length of `users` is changed,
or one of `firstName` is changed.
It keeps returning a cached result even if `lastName` is changed.

### Closing notes

What inspired me to develop this was the relationship between MobX and Immer.
I'm not familiar with their implementations at all, but
it feels me like Immer is a subset of MobX for broader use cases.
I wanted to create something like Immer. Immer lets you magically
convert mutable (write) operations to immutable objects.
proxy-memoize lets you magically create selector (read) functions
for immutable objects.
