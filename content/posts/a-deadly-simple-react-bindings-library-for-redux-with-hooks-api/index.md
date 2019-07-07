---
title: "A deadly simple React bindings library for Redux with Hooks API"
subtitle: "Love Redux and Hooks API."
date: 2018-11-16T12:00:00+09:00
tags: ["react", "hooks", "typescript", "global-state", "redux", "proxy"]
---

> There is a follow-up article which describes the implementation details of this library. Please visit [here](https://blog.axlight.com/posts/developing-react-custom-hooks-for-redux-without-react-redux/).

Redux is one of the popular libraries for React. Although Redux itself is not tied to React, it's often used with the official bindings `react-redux` that allows to connect Redux state to any React components.

In this article, we propose another bindings library to combine React and Redux. This library tries to be simple and easy for beginners. As of writing, this is still an experimental project and feedbacks are welcome.

### Typical react-redux usage

Let's assume we have a simple global state.

```javascript
const initialState = {
  counter: 0,
  person: {
    firstName: 'Harry',
    lastName: 'Potter',
    age: 11,
  },
};
```

The component that shows the person name would be something like this.

```jsx
const PersonName = ({ firstName, lastName }) => (
  <div>
    <span>{firstName}</span>
    <span>{lastName}</span>
  </div>
);
const mapStateToProps = state => ({
  firstName: state.person.firstName,
  lastName: state.person.lastName,
});
export default connect(mapStateToProps)(PersonName);
```

With the proposal of React Hooks API, react-redux seeks a way to support the API [[1](https://github.com/reduxjs/react-redux/issues/1063)]. One possible way would be something like the following.

```jsx
const PersonName = () => {
  const mapStateToProps = state => ({
    firstName: state.person.firstName,
    lastName: state.person.lastName,
  });
  const [{ firstName, lastName }] = useRedux(mapStateToProps);
  return (
    <div>
      <span>{firstName}</span>
      <span>{lastName}</span>
    </div>
  );
};
export default PersonName;
```

In either cases, `mapStateToProps` is important to detect if the part of the state is changed. (Technically, in the hooks example code above, the rendering process won't stop even if `useRedux` detects no change. We would need proper memoization or bailing out rendering.)

### react-hooks-easy-redux

Especially for beginners, often writing `mapStateToProps` is not trivial, and it may include expensive computation by chance. We would like to eliminate this problem by completely removing it.

react-hooks-easy-redux is a library we are developing. It's still experimental, but you can try it out. Note: This is based on unreleased Hooks API, and it is not for production yet.

https://github.com/dai-shi/react-hooks-easy-redux

The example code above would look like the following with this library.

```jsx
import { useReduxState } from 'react-hooks-easy-redux';
const PersonName = () => {
  const state = useReduxState();
  return (
    <div>
      <span>{state.person.firstName}</span>
      <span>{state.person.lastName}</span>
    </div>
  );
};
export default PersonName;
```

This works magically by Proxy, and even if the counter property or the age property in the state is changed, the rendering won't occur.

You can try some examples and also write your own example.

```bash
git clone https://github.com/dai-shi/react-hooks-easy-redux.git
cd react-hooks-easy-redux
npm install
npm run examples:minimal
```

You can change "minimal" to other examples like "typescript" and "deep".

You can also try it online.

https://codesandbox.io/s/github/dai-shi/react-hooks-easy-redux/tree/master/examples/01_minimal

### Final notes

Some of readers may have noticed that there's no discussion about actions or `mapDispatchToProps` in this article. This is because our library only supports exposing `dispatch`, and we want to focus the discussion. Even with `react-redux`, we would recommend beginners to use "dispatch as a prop" for learning which could be more intuitive.

### Changelog

- _[2018-11-17]:_ Initial publication.
- _[2018-12-14]:_ Removed bailOutHack and added [a follow-up article](https://blog.axlight.com/posts/developing-react-custom-hooks-for-redux-without-react-redux/).
