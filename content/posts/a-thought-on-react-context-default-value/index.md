---
title: "A thought on React Context default value"
subtitle: "How do you like this idea?"
date: 2019-01-11T12:00:00+09:00
tags: ["react", "context", "hooks"]
---

### Introduction

It's been about a year since React Context API is out. This is powerful and it's even more powerful with upcoming React Hooks API, namely `useContext`. Please check out official documents to learn them in detail.

https://reactjs.org/docs/context.html

https://reactjs.org/docs/hooks-intro.html

Now, this very short article is just about the default value of a context. When you create a context, you pass a default value in the first argument.

> The `defaultValue` argument is **only** used when a component does not have a matching Provider above it in the tree. This can be helpful for testing components in isolation without wrapping them. Note: passing `undefined` as a Provider value does not cause consuming components to use `defaultValue`.

So, this is only used if there's no Provider. It's often the case that we require a Provider and if it doesn't exist, we warn developers.

### Typical solution

In various code, what I see is something like the following:

```javascript
import { createContext, useContext } from 'React';
const MyContext = createContext(null);
export const MyProvider = MyContext.Provider;
export const useMyValueFoo = () => {
  const value = useContext(MyContext);
  if (!value) {
    throw new Error('You probably forgot to put <MyProvider>.');
  }
  return value.foo;
};
```

The intention behind this code is to let developers notice if they forget to put Provider in the component tree.

### My proposal

I thought what if we set this warning object in the default value. This would only work if we use an object value for a context, but here's the code:

```javascript
import { createContext, useContext } from 'React';
const warningObject = {
  get foo() {
    throw new Error('You probably forgot to put <MyProvider>.');
  },
};
const MyContext = createContext(warningObject);
export const MyProvider = MyContext.Provider;
export const useMyValueFoo = () => {
  const value = useContext(MyContext);
  return value.foo;
};
```

Again, this only works for an object value. I'm not sure other possible drawbacks. Maybe, it's hard to add a type annotation?

I've used it in my project. If you are interested, please check out how it's really used in the repository.

https://github.com/dai-shi/react-hooks-easy-redux

### Final notes

React hooks is still in alpha, but we are expecting it soon. Meanwhile, we look for its possibilities. Feedbacks are always welcome.
