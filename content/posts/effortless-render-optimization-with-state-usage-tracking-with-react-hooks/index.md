---
title: "Effortless render optimization with state usage tracking with React hooks"
subtitle: "Try react-tracked and reactive-react-redux"
date: 2019-07-21T13:00:00+09:00
tags: ["react", "context", "hooks", "redux", "proxy", "global-state"]
---

### Introduction

React useContext is very handy to avoid prop drilling.
It can be used to define global state or shared state
that multiple components in the tree can access.

However, useContext is not specifically designed for
global state and there's a caveat.
Any change to context value propagates all useContext
to re-render components.

This post shows some example code about the problem
and the solution with state usage tracking.

### Problem

Let's assume a person object as a state.

```javascript
const initialState = {
  firstName: 'Harry',
  familyName: 'Potter',
};
```

We use a context and a local state.

```jsx
const PersonContext = createContext(null);

const PersonProvider = ({ children }) => {
  const [person, setPerson] = useState(initialState);
  return (
    <PersonContext.Provider value={[person, setPerson]}>
      {children}
    </PersonContext.Provider>
  );
};
```

Finally, here's a component to display the first name of the person.

```jsx
const DisplayFirstName = () => {
  const [person] = useContext(PersonContext);
  return (
    <div>First Name: {person.firstName}</div>
  );
};
```

So far, so good. However, the problem is when you
update the family name of the person and keep the first name same.
It will trigger `DisplayFirstName` to re-render,
even the render result is the same.

Please note this is not really a problem, until it becomes a problem.
Typically, most smaller apps just work, but some bigger apps
might have performance issues.

### Solution: state usage tracking

Let's see how state usage tracking solves this.

The provider looks a bit different, but essentially the same.

```jsx
const usePerson = () => useState(initialState);
const { Provider, useTracked } = createContainer(usePerson);

const PersonProvider = ({ children }) => (
  <Provider>
    {children}
  </Provider>
);
```

The `DisplayFirstName` component will be changed like this.

```jsx
const DisplayFirstName = () => {
  const [person] = useTracked();
  return (
    <div>First Name: {person.firstName}</div>
  );
};
```

Notice the change?
Only the difference is `useTracked()` instead of `useContext(...)`.

With this small change, state usage in `DisplayFirstName` is tracked.
And now even if the family name is updated,
this component will not re-render as long as the first name is not updated.

This is effortless render optimization.

### Advanced Example

Some readers might think
this can also be accomplished by `useSelector`-like hooks.

Here's another example in which `useTracked` is much easier.

```javascript
const initialState = {
  firstName: 'Harry',
  familyName: 'Potter',
  showFullName: false,
};
```

Suppose we have a state like the above,
and let's create a component with a condition.

```jsx
const DisplayPersonName = () => {
  const [person] = useTracked();
  return (
    <div>
      {person.showFullName ? (
        <span>
          Full Name: {person.firstName}
          <Divider />
          {person.familyName}
        </span>
      ) : (
        <span>First Name: {person.firstName}</span>
      )}
    </div>
  );
};
```

This component will re-render either in two scenarios.

- a) when `firstName` or `familyName` is updated, if showing full name
- b) when `firstName` is updated, if not showing full name

Reproducing the same behavior with `useSelector` would
not be easy and probably end up with separating components.

### Projects using state usage tracking

There are two projects using state usage tracking.

##### reactive-react-redux

https://github.com/dai-shi/reactive-react-redux

This is an alternative library to react-redux.
It has the same hooks API and `useTrackedState` hook.

##### react-tracked

https://github.com/dai-shi/react-tracked

This is a library without Redux dependency.
The example in this post is based on this.
It has a compatible hooks API with reactive-react-redux.

## Closing notes

This post focused on how state usage tracking can easily be used.
We didn't discuss about implementation of these libraries.

Technically, there are two hurdles.
In short, we use Proxy API to track the state usage.
We also use an undocumented feature in Context API
to stop propagation.
If you are interested in those internals,
please check out those GitHub repositories.
