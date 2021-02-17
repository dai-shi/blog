---
title: "4 options to prevent extra rerenders with React context"
subtitle: "How do you like react-tracked"
date: 2019-08-21T21:00:00+09:00
tags: ["react", "context", "hooks", "global-state"]
---

### Introduction

React context and useContext are very handy.
You would have no problem using it while developing a small app.
If the size of your app became non-trivial,
you might experience some performance issues with regard to useContext.
This is because useContext will trigger rerender
whenever the context value is changed.
This happens even if the part of the value is not used in render.
This is by design. If useContext were to conditionally
trigger rerenders, the hook would become non-composable.

There has been several discussions, especially in [this issue](https://github.com/facebook/react/issues/14110).
Currently, there's no direct solution from React core.
Three options are described in [this issue](https://github.com/facebook/react/issues/15156#issuecomment-474590693).

This post shows an example with these three options and
another option with a library called [react-tracked](https://github.com/dai-shi/react-tracked).

### Base example

Let's have a minimal example: A person object with `firstName` and `familyName`.

```javascript
const initialState = {
  firstName: 'Harry',
  familyName: 'Potter',
};
```

We define a reducer to feed into useReducer.

```javascript
const reducer = (state, action) => {
  switch (action.type) {
    case 'setFirstName':
      return { ...state, firstName: action.firstName };
    case 'setFamilyName':
      return { ...state, familyName: action.familyName };
    default:
      throw new Error('unexpected action type');
  }
};
```

Our context provider looks like this.

```jsx
const NaiveContext = () => {
  const value = useReducer(reducer, initialState);
  return (
    <PersonContext.Provider value={value}>
      <PersonFirstName />
      <PersonFamilyName />
    </PersonContext.Provider>
  );
};
```

`PersonFirstName` is implemented like this.

```jsx
const PersonFirstName = () => {
  const [state, dispatch] = useContext(PersonContext);
  return (
    <div>
      First Name:
      <input
        value={state.firstName}
        onChange={(event) => {
          dispatch({ type: 'setFirstName', firstName: event.target.value });
        }}
      />
    </div>
  );
};
```

Similar to this, `PersonFamilyName` is implemented.

So, what would happen is if `familyName` is changed,
`PersonFirstName` will rerender resulting in the same output as before.
Because users wouldn't notice the change, this wouldn't be a big problem.
But, it may slow down when the number of components to rerender is large.

Now, how to solve this? Here are 4 options.

### Option 1: Split contexts

The most preferable option is to split contexts.
In our example, it will be like this.

```javascript
const initialState1 = {
  firstName: 'Harry',
};

const initialState2 = {
  familyName: 'Potter',
};
```

We define two reducers and use two contexts.
If this makes sense in your app, it's always recommended in idiomatic React.
But if you need to keep them in a single state, you can't take this option.
Our example is probably so, because it's meant to be a single person object.

### Option 2: React.memo

The second option is to use React.memo.
I think this is also idiomatic.

We don't change the context in the base example.
`PersonFirstName` is re-implemented with two components.

```jsx
const InnerPersonFirstName = React.memo(({ firstName, dispatch }) => (
  <div>
    First Name:
    <input
      value={firstName}
      onChange={(event) => {
        dispatch({ type: 'setFirstName', firstName: event.target.value });
      }}
    />
  </div>
);

const PersonFirstName = () => {
  const [state, dispatch] = useContext(PersonContext);
  return <InnerPersonFirstName firstName={state.firstName} dispatch={dispatch} />;
};
```

When `familyName` in the person object is changed,
`PersonFirstName` rerenders.
But, `InnerPersonFirstName` doesn't rerender because `firstName` isn't changed.

All complex logic is moved into `InnerPersonFirstName`
and `PersonFirstName` is typically lightweight.
So, performance wouldn't be an issue with this pattern.

### Option 3: useMemo

If React.memo doesn't work as you expect, you can useMemo as the third option.
I wouldn't personally recommend this.
There might be some limitations. For example, you can't use hooks.

`PersonFirstName` looks like this with useMemo.

```jsx
const PersonFirstName = () => {
  const [state, dispatch] = useContext(PersonContext);
  const { firstName } = state;
  return useMemo(() => {
    return (
      <div>
        First Name:
        <input
          value={firstName}
          onChange={(event) => {
            dispatch({ type: 'setFirstName', firstName: event.target.value });
          }}
        />
      </div>
    );
  }, [firstName, dispatch]);
};
```

### Option 4: react-tracked

The fourth option is to use a library.

<https://github.com/dai-shi/react-tracked>

With this library, our provider would look a bit different like this.

```jsx
const { Provider, useTracked } = createContainer(() => useReducer(reducer, initialState));

const ReactTracked = () => {
  return (
    <Provider>
      <PersonFirstName />
      <PersonFamilyName />
    </Provider>
  );
};
```

`PersonFirstName` is implemented like this.

```jsx
const PersonFirstName = () => {
  const [state, dispatch] = useTracked();
  return (
    <div>
      First Name:
      <input
        value={state.firstName}
        onChange={(event) => {
          dispatch({ type: 'setFirstName', firstName: event.target.value });
        }}
      />
    </div>
  );
};
```

Notice the change from the base example.
It's just one line change.

```diff
-  const [state, dispatch] = useContext(PersonContext);
+  const [state, dispatch] = useTracked();
```

How does this work?
The state returned by `useTracked()` is wrapped by Proxy,
and its usage is tracked.
It means that the hook knows
that only the `firstName` property is used in render.
This allows to only trigger rerender
when used properties are changed.
This effortless optimization is what I call "state usage tracking."

### What is state usage tracking

For more information, please visit my other blog posts. For example:

[What is state usage tracking? A novel approach to intuitive and performant global state with React hooks and Proxy](https://blog.axlight.com/posts/what-is-state-usage-tracking-a-novel-approach-to-intuitive-and-performant-api-with-react-hooks-and-proxy/)

There's also [a list of blog posts](https://github.com/dai-shi/react-tracked#blogs).

### Full example demo

<iframe src="https://codesandbox.io/embed/github/dai-shi/react-tracked/tree/master/examples/08_comparison?fontsize=14" title="react-tracked-example" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

[Source code in the repo](https://github.com/dai-shi/react-tracked/tree/master/examples/08_comparison)

### Closing notes

If you have already read some previous blog posts of mine,
there can be no new findings in this post.

I would like to learn more coding patterns from others.
Please let me know how it would look like in your use case.
