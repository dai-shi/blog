---
title: "New React Redux coding style with hooks without selectors"
subtitle: "Use custom hooks instead of selectors"
date: 2019-04-20T12:00:00+09:00
tags: ["react", "hooks", "redux", "proxy"]
---

### Introduction

React Redux is one of popular web tech stacks. As I found React hooks so promising, I have been developing a hooks-based library for React Redux called "reactive-react-redux."

https://github.com/dai-shi/reactive-react-redux

Coding style using this library is mentally different from the official react-redux. Thanks to Proxy, developers don't need to care about optimization, but just focus on composition of logic.

This library provides two basic hooks: `useReduxState` and `useReduxDispatch`. This article shows example code how to use them.

### Preliminary

We use a tiny example in this article. The following is how our Redux store state looks like.

```javascript
const state = {
  items: [
    { title: 'ttt1', description: 'ddd1' },
    { title: 'ttt2', description: 'ddd2' },
    { title: 'ttt3', description: 'ddd3' },
  },
};
```

### How to useReduxState

We can create a custom hook based on `useReduxState`. Let's create a hook that receives an item index and returns an item object.

```javascript
const useItem = (index) => {
  const state = useReduxState();
  return state.items[index];
};
```

Now, we use this hook to create a component.

```jsx
const Item = ({ index }) => {
  const item = useItem(index);
  return (
    <div>{item.title}{' - '}{item.description}</div>
  );
};
```

This style is totally different from the one with `connect`, which would look like the following.

```jsx
const Item = ({ item }) => (
  <div>{item.title}{' - '}{item.description}</div>
);
const mapStateToProps = (state, ownProps) => ({
  item: state.items[ownProps.index],
});
const ConnectedItem = connect(mapStateToProps)(Item);
```

### Effortless optimization with useReduxState

Suppose our component only uses `title` and no `description`, our component would be something like this.

```jsx
const ItemWithTitle = ({ index }) => {
  const item = useItem(index);
  return (
    <div>{item.title}</div>
  );
};
```

Because `useReduxState` tracks the usage of state in render, it won't trigger re-render if only description is changed. We don't need to create "useItemWithTitle" and "useItemWithTitleAndDescription" hooks, but just one "useItem" does the job.

This is an important aspect of the hook. We can even create "useItemTitle."

```javascript
const useItemTitle = (index) => {
  const item = useItem(index);
  return item.title;
};
```

Note that the examples above can be over-simplified and they don't seem very useful, but hopefully they explain the idea.

### How to useReduxDispatch

There can be various patterns for useReduxDispatch.

Probably the simplest is to create a hook to dispatch a specific action.

```javascript
const useAddItem = () => {
  const dispatch = useReduxDispatch();
  const addItem = useCallback(title => dispatch({ type: 'ADD_ITEM', title }, [dispatch]);
  return addItem;
};
```

If you already have an action creator, you can create a general hook.

```javascript
const useAction = (actionCreator) => {
  const dispatch = useReduxDispatch();
  return useCallback((...args) => dispatch(actionCreator(...args)), [dispatch];
};
```

You could also pass action creators, if you prefer.

```javascript
const useActions = (actionCreators) => {
  // ...
};
```

## Final thoughts

When I started writing this article, I was hoping to come up with better example code so that readers get aha. Reading the entire article again, I'm not very confident if this is appealing as expected. I decided to let go anyway, and will see. Comments would be appreciated by [Twitter](https://twitter.com/dai_shi) or [GitHub issues](https://github.com/dai-shi/reactive-react-redux/issues).
