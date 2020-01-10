---
title: "Lazy Load Apollo Link in Apollo Client"
subtitle: "10 lines of code to help"
date: 2020-01-10T23:00:00+09:00
tags: ["apollo", "graphql", "apollo-link", "lazy-loading"]
---

## Introduction

This is a short post about my small library.

[Apollo Client](https://github.com/apollographql/apollo-client)
is a library for GraphQL.
[Apollo Link](https://github.com/apollographql/apollo-link)
is an interface to extend Apollo Client.

Typically, you would initialize apollo client like this.

```js
import { ApolloClient } from 'apollo-client';
import { InMemoryCache } from 'apollo-cache-inmemory';
import { HttpLink } from 'apollo-link-http';

const cache = new InMemoryCache();
const link = new HttpLink({ uri });

const client = new ApolloClient({
  cache: cache,
  link: link,
});
```

I want to define the link in another file and lazy load it,
because it is not an HttpLink but a complicated large link.

## How to use

We use [dynamic imports](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import) for this.

Let's assmume we have `link.js` file that exports an apollo link.
It'd be nice to dynamic import it.

```js
import { lazy } from 'apollo-link-lazy';

const link = lazy(() => import('./link'));
```

`import()` returns a promise, but there's no `await`.
How is this possible?

## How to implement

Interestingly, Apollo Link is asynchronous by nature.
However, it's not promise based. It has observable interface.

So, all you need is to covert a promise into an observable.

Here's the code.

```js
import { ApolloLink, fromPromise, toPromise, Observable } from 'apollo-link';

export const lazy = (factory) => new ApolloLink(
  (operation, forward) => fromPromise(
    factory().then((resolved) => {
      const link = resolved instanceof ApolloLink ? resolved : resolved.default;
      return toPromise(link.request(operation, forward) || Observable.of());
    }),
  ),
);
```

Luckily, apollo-client exports `fromPromise` and `toPromise`
utility functions. Hence, it can be implemented so easily.

A little trick here is to support both ApolloLink promises and default exports.

## Demo

I developed this code as a library.

<https://github.com/dai-shi/apollo-link-lazy>

You can install it and use it. It supports TypeScript.

Here's also a demo in codesandbox.

<https://codesandbox.io/s/github/dai-shi/apollo-link-lazy/tree/master/examples/02_typescript>

## Closing notes

As my motivation was code splitting, supporting default exports like
[React.lazy](https://reactjs.org/docs/code-splitting.html#reactlazy)
was actually enough.
Because it also supports direct promises, we can use it for any
async initialization like the following.

```js
import { lazy } from 'apollo-link-lazy';

const link = lazy(async () => {
  // await ...
  return new ApolloLink(...);
});
```

I hope this may help other developers who try lazy loading of apollo links.
