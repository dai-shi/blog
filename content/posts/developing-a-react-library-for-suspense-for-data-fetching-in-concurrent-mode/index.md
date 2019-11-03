---
title: "Developing a React Library for Suspense for Data Fetching in Concurrent Mode"
subtitle: "A new experimental react-hooks-fetch"
date: 2019-11-03T21:00:00+09:00
tags: ["react", "hooks", "fetch", "suspense", "concurrent"]
---

## Introduction

We have been waiting for "Suspense for Data Fetching" for a long time.
It is now provided as an experimental feature in
[the experimental channel](https://reactjs.org/blog/2019/10/22/react-release-channels.html).

Check out the official docs for details.

1. [Introducing Concurrent Mode](https://reactjs.org/docs/concurrent-mode-intro.html)
2. [Suspense for Data Fetching](https://reactjs.org/docs/concurrent-mode-suspense.html)
3. [Concurrent UI Patterns](https://reactjs.org/docs/concurrent-mode-patterns.html)
4. [Adopting Concurrent Mode](https://reactjs.org/docs/concurrent-mode-adoption.html)
5. [Concurrent Mode API Reference](https://reactjs.org/docs/concurrent-mode-reference.html)

They are trying best to explain new mind sets with analogies.
That means it's totally different from the usage with traditional React.
Yes, it is different and promising.

This post is to explore a usage with Suspense for Data Fetching.
Please note that all features are experimental and
the current understanding could be wrong in the future.

To get the benefit of Suspense for Data Fetching in Concurrent Mode,
we should use the "Render-as-You-Fetch" pattern.
This requires to start fetching before rendering.
We need to have new mental model because we are so used to fetching
in useEffect or componentDidMount.

This post doesn't provide any specific answer to best practices yet,
but here's what I did now.

## createFetcher

Let's create a "fetcher" that wraps a fetch function.
This can be an arbitrary async function that returns a Promise.

```js
const fetcher = createFetcher(async url => (await fetch(url)).json());
```

This is a general GET fetcher that takes a url as an input
and assumes a JSON response.
Typically, we'd want to create more specialized fetchers.

A fetcher provides two methods: `prefetch` and `lazyFetch`.

If you invoke `prefetch`, it will start the fetch function
and you will get a "suspendable."
A "suspendable" is an object with two properties: `data` and `refetch`.
The `data` is to get the promise result, but it will throw a promise
if the promise is not resolved.
The `refetch` will run the fetch function again and returns a new "suspendable."

If you invoke `lazyFeth`, you will get a "suspendable"-like,
with fallback data and a lazy flag.
It will actually never suspend, but you can treat it as a "suspendable"
just like the one returned by `prefetch`.

Now, the TypeScript typing of createFetcher is the following:

```ts
type Suspendable<Data, Input> = {
  data: Data;
  refetch: (input: Input) => Suspendable<Data, Input>;
  lazy?: boolean;
};

type Fetcher<Data, Input> = {
  prefetch: (input: Input) => Suspendable<Data, Input>;
  lazyFetch: (fallbackData: Data) => Suspendable<Data, Input>;
};

export const createFetcher: <Data, Input>(
  fetchFunc: (input: Input) => Promise<Data>,
) => Fetcher<Data, Input>;
```

The implementation of this is a bit long.

```js
export const createFetcher = (fetchFunc) => {
  const refetch = (input) => {
    const state = { pending: true };
    state.promise = (async () => {
      try {
        state.data = await fetchFunc(input);
      } catch (e) {
        state.error = e;
      } finally {
        state.pending = false;
      }
    })();
    return {
      get data() {
        if (state.pending) throw state.promise;
        if (state.error) throw state.error;
        return state.data;
      },
      get refetch() {
        return refetch;
      },
    };
  };
  return {
    prefetch: input => refetch(input),
    lazyFetch: (fallbackData) => {
      let suspendable = null;
      const fetchOnce = (input) => {
        if (!suspendable) {
          suspendable = refetch(input);
        }
        return suspendable;
      };
      return {
        get data() {
          return suspendable ? suspendable.data : fallbackData;
        },
        get refetch() {
          return suspendable ? suspendable.refetch : fetchOnce;
        },
        get lazy() {
          return !suspendable;
        },
      };
    },
  };
};
```

The use of `prefetch` is almost always preferred.
The `lazyFetch` is only provided as a workaround
for the "Fetch-on-Render" pattern.

Once you get a "suspendable,"
you can use it in render and React will take care of the rest.

Because we need to invoke `prefetch` before creating a React element.
we could only do it outside of render functions.
As of writing, we do it in the component file globally,
assuming we know what we want as an initial "suspendable."
This would probably make testing difficult.

## useSuspendable

The fetcher created by `createFetcher` is functionally complete,
but it would be nice to have handy hooks to use "suspendable"s.

The simplest one is `useSuspendable`.
It simply stores a single "suspendable" in a local state.

The implementation of `useSuspendable` is the following.

```js
export const useSuspendable = (suspendable) => {
  const [result, setResult] = useState(suspendable);
  const origFetch = suspendable.refetch;
  return {
    get data() {
      return result.data;
    },
    refetch: useCallback((nextInput) => {
      const nextResult = origFetch(nextInput);
      setResult(nextResult);
    }, [origFetch]),
    lazy: result.lazy,
  };
};
```

The result returned by the useSuspendable hook is
almost like a normal "suspendable" but with a slight difference.
If you invoke `refetch`, instead of returning a new "suspendable,"
it will replace the state value with the new "suspendable."

## The library

I've developed the above code into a library.

<https://github.com/dai-shi/react-hooks-fetch>

This is highly experimental and it will change.

## Demo

Here's one small example using this library.

[Codesandbox](https://codesandbox.io/s/github/dai-shi/react-hooks-fetch/tree/master/examples/02_typescript)

There are some other examples in the repo.

## Closing notes

I hesitated a bit to write a post like this, which is highly experimental
and can even change in a couple of days after writing.
Nevertheless, I would like the community to try the new world
with Suspense for Data Fetching and give some feedbacks.
