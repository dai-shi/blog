---
title: "Diving Into React Suspense Render-as-You-Fetch for REST APIs"
subtitle: "Obsolete useEffect based data fetching"
date: 2019-12-16T22:00:00+09:00
tags: ["react", "async", "fetch", "suspense", "concurrent"]
---

### Introduction

React released Concurrent Mode in the experimental channel
and [Suspense for Data Fetching](https://reactjs.org/docs/concurrent-mode-suspense.html).
This release is for library authors, and not for production apps yet.
The new data fetching pattern proposed is called Render-as-You-Fetch.

This post mainly discuss Render-as-You-Fetch for basic fetch calls,
like calling REST APIs. But, some of discussions are not limited to REST.
One could invoke GraphQL endpoints with simple fetch calls.
For more complex use cases with GraphQL, it's worth looking into
[Relay documentation](https://relay.dev/docs/en/experimental/a-guided-tour-of-relay#advanced-data-fetching) too.

### Problems with useEffect based data fetching

Let's first discuss the problems with the typical solution,
which is to start data fetching in useEffect.

#### Too many loading indicators

Typical useEffect based data fetching is like this.

```jsx
const Component = () => {
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  useEffect(() => {
    (async () => {
      setLoading(true);
      setResult(await fetchData());
      setLoading(false);
    })();
  }, []);
  // ...
};
```

If we use this pattern in various components,
users end up seeing lots of loading indicators in their screen.

We could solve this issue, by having one loading counter in a parent
component and share it among child components.

The Suspense component is a native solution to this issue.

#### Fetch calls run too late

In the above example, `fetchData` runs in useEffect.
It runs only after all components are painted on a browser.
That may or may not be very late depending on applications.

This is crucial when using `React.lazy`.
Fetch calls can only invoked only after components are loaded.

We'd want starting a fetch call and loading a component at the same time.

#### Fetch calls waterfall

Because of the timing described above,
there is a specific behavior called "waterfall."
If a parent component is in a loading state, a child component
will not render and thus will not start a fetch call in useEffect.
Only when a fetch call in the parent component is finished,
the fetch call in the child component can start.

Please also refer [the React documentation](https://reactjs.org/docs/concurrent-mode-suspense.html#approach-1-fetch-on-render-not-using-suspense)
for an example about waterfall.

#### Troublesome useEffect deps / useCallback

It's recommended to put props that are used in useEffect
to deps of the useEffect second argument.
For some reason, if you need to create a function in advance,
that should be wrapped by useCallback.

The typical custom hook is like this.

```jsx
const useFetch = (fetchFunc) => {
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  useEffect(() => {
    (async () => {
      setLoading(true);
      setResult(await fetchFunc());
      setLoading(false);
    })();
  }, [fetchFunc]);
  return { loading, result };
};

const Component = ({ id }) => {
  const fetchFunc = useCallback(async () => {
    // fetch with id
  }, [id]);
  const { loading, result } = useFetch(fetchFunc);
  // ...
};
```

This pattern is not very easy for beginners.
It can be said that useEffect is overused for data fetching,
or more precisely there has been no other means until Suspense lands.

### Mental model with React Suspense

Render-as-You-Fetch requires a new mental model.
Otherwise, it's hard to understand the library for the new pattern.
Here's some random points to understand the new pattern.

#### Don't useEffect

Don't think remote data as an effect of props.
Create it at the same time when elements are created.

Pseudo code is something like this.

```jsx
const fetchRemoteData = ...;
const Component = ...;

const remoteData = fetchRemoteData();
<Component remoteData={remoteData} />
```

#### Pass remote data as props or store in state

Pass fetching data as props along with its dependent props.

Pseudo code is something like this.

```jsx
const Component = ({ useId, userData }) => {
  // userData is remote data fetched with `userId`
  // ...
};
```

Or, keep it in state directly.

```jsx
const Component = () => {
  const [userId, setUserId] = useState();
  const [userData, setUserData] = useState();
  // Set userId and userData at the same time. Not as dependencies.
  // Typically done in callbacks.
  // ...
};
```

#### Treat remote data just like local data

Thanks to Suspense, the render code doesn't need to care
if data is locally available or fetching remotely.
You can just use it.

Pseudo code is something like this.

```jsx
const Component = ({ localData, remoteData }) => (
  <div>
    <div>Local Name: {localData.name}</div>
    <div>Remote Name: {remoteData.name}</div>
  </div>
);
```

### Use cases of Render-as-You-Fetch

Now, let's think about how we use Render-as-You-Fetch pattern
if we have a good library.

We assume we had a library that allow to create a suspendable result,
which can be used just like local data.
That means, if the result is not ready it will throw a promise.

#### Single fetch

The simplest example is just one fetch call.

```jsx
// Define component
const Component = ({ result }) => <div>Name: {result.name}</div>;

// Create a suspendable result
const result = prefetch(async () => (await fetch('https://swapi.co/api/people/1/')).json());

// Create a React element
<Component result={result} />
```

#### Multiple fetch

If we need to run two fetch calls in parallel,
we create them at the same time.

```jsx
// Define component
const Component = ({ result }) => <div>Name: {result.name}</div>;

// Create two suspendable results
const result1 = prefetch(async () => (await fetch('https://swapi.co/api/people/1/')).json());
const result2 = prefetch(async () => (await fetch('https://swapi.co/api/people/2/')).json());

// Create a React element
<div>
  <Component result={result1} />
  <Component result={result2} />
</div>
```

It totally depends on how you put `<Suspense>` in the tree
whether the result is shown at once or one by one.

Check out [the API documentation](https://reactjs.org/docs/concurrent-mode-reference.html#suspensecomponent)
to learn more about how to use Suspense and SuspenseList.

#### Dynamic fetch

Data fetching is not always static,
we may need to fetch data dynamically.
For example, if a user click a button to re-run fetch,
we need a state like this.

```jsx
const Component = () => {
  const [result, setResult] = useState({});
  const onClick = () => {
    const nextId = 1 + Math.floor(Math.random() * 10);
    const nextResult = prefetch(async () => (await fetch(`https://swapi.co/api/people/${nextId}/`)).json());
    setResult(nextResult);
  };
  return (
    <div>
      <div>Name: {result.name}</div>
      <button type="button" onClick={onClick}>Refetch</button>
    </div>
  );
};
```

This is an example of prefetching in a callback,
but this pattern could apply to all non-React callbacks.
Just simply take it as feeding suspendable results into React tree.

#### Incremental fetch

If two fetch calls are dependent and we want to show
the intermediate state to a user, we need incremental loading.

```jsx
// Define component
const Person = ({ person }) => <div>Person Name: {person.name}</div>;
const Films = ({ films }) => (
  <ul>
    {films.map(film => (
      <li key={film.url}>Film Title: {film.title}</li>
    ))}
  </ul>
);

// Create two suspendable results
const person = prefetch(async () => (await fetch('https://swapi.co/api/people/1')).json());
const films = prefetch(
  urls => Promise.all(urls.map(async url => (await fetch(url)).json())),
  person => person.films,
  person,
);

// Create a React element
<Suspence fallback={<div>Loading...</div>}>
  <Person person={person} />
  <Suspense fallback={<div>Loading films...</div>}>
    <Films films={films} />
  </Suspense>
</Suspense>
```

This shows "Person Name" as soon as it's available,
and shows "Loading films..." until they are ready.

It requires a trick to make this work.
The function `person => person.films` in prefetch can suspend
just like React render can suspend.
Otherwise, we don't know when to start fetching films.

### Use of proxies

If we want to treat remote data like local data,
it's important to avoid indirection.
[Proxy](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Proxy) allows such interface.

With Proxy, we can do like the following.

```javascript
console.log(result.name); // throws a promise until it's resolved

console.log(result.name); // works as expected after that
```

### Notes for caching

It's important how we deal with caching.
Our current approach is that we don't provide global cache.
Caching is a hard problem.
Instead, we simply store results like normal data.
It's very intuitive and works well for simple use cases.

For complex caching approaches with data normalization,
check out various projects.

- [Apollo Client](https://www.apollographql.com/docs/react/)
- [SWR](https://swr.now.sh)
- [Relay](https://relay.dev)

### Experimental projects

What we described above is not a dream,
we have been developing some experimental libraries.
They are ongoing projects and will not reflect
what is described in this post in the future.

#### react-suspense-fetch

<https://github.com/dai-shi/react-suspense-fetch>

This project provides `prefetch` that is described above.
Its implementation is actually nothing to do with React,
but it follows the convention of throwing promises.

Note that the API can change soon.

#### react-hooks-fetch

<https://github.com/dai-shi/react-hooks-fetch>

This project is to provide hooks for React Suspense.
While not currently, it will be based on react-suspense-fetch.

The API will also change soon.

### Closing notes

Render-as-You-Fetch is totally a new pattern and
useEffect based data fetching will be obsoleted.
It's uncertain if this post can give enough insights about that.
It would be nice if many developers discuss about this topic
and come up with various ideas and use cases.
