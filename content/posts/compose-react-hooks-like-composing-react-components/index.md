---
title: "Compose React hooks like composing React components"
subtitle: "Why custom hooks are so powerful"
date: 2019-04-18T12:00:00+09:00
tags: ["react", "hooks"]
---

This short article shows code examples how hooks are analogous to components in terms of composition. Without a word, let's jump in the code.

### React components

Let's use a minimalistic example. Here's a component.

```jsx
const Person = ({ person }) => (
  <div>
    <div className="personFirstName">
      <span>First Name:<span>
      <span>{person.firstName}</span>
    </div>
    <div className="personLastName">
      <span>Last Name:<span>
      <span>{person.lastName}</span>
    </div>
  </div>
);
```

Well, this component is a little bit big, so let's split it into functions and compose them.

```jsx
const FirstName = ({ name }) => (
  <div className="personFirstName">
    <span>First Name:<span>
    <span>{name}</span>
  </div>
);
const LastName = ({ name }) => (
  <div className="personLastName">
    <span>Last Name:<span>
    <span>{name}</span>
  </div>
);
const Person = ({ person }) => (
  <div>
    <FirstName name={person.firstName} />
    <LastName name={person.lastName} />
  </div>
);
```

Now, it looks organized. Why do we do this? Because, we can then use `FirstName` component in other components. It's reusability.

### React hooks

What about hooks? Let's suppose we have `usePerson` hook, which returns the person object. It's not important how it's implemented, but for example, we can assume it's with `useContext` as a simple concrete case.

```jsx
const Person = () => {
  const person = usePerson();
  const firstName = person.firstName;
  const lastName = person.lastName;
  return <div>{firstName}{' '}{lastName}</div>
};
```

This is over-simplified, but let us split it into functions and compose them.

```jsx
const useFirstName = () => {
  const person = usePerson();
  return person.firstName;
};
const useLastName = () => {
  const person = usePerson();
  return person.lastName;
};
const Person = () => {
  const firstName = useFirstName();
  const lastName = useLastName();
  return <div>{firstName}{' '}{lastName}</div>;
};
```

I hope you get the idea. Now, you can reuse `useFirstName` hook in other components.

### Conclusion

Composition matters. A component can be built with other components, so can a hook be built with other hooks.

----

### Appendix

We have `children` in components. What would be an equivalent in hooks? I'm not 100% certain about this, but it might be hooks factory. Or, it's just one variant of it. Here's the code for the example above.

```jsx
const createPersonHooks = (usePerson) => {
  const useFirstName = () => {
    const person = usePerson();
    return person.firstName;
  };
  const useLastName = () => {
    const person = usePerson();
    return person.lastName;
  };
  return { useFirstName, useLastName };
};
```

Again, I'm not so sure about it, but just an idea. [Constate](https://github.com/diegohaz/constate) seems to take a similar approach.

----

Comments would be appreciated by [Twitter](https://twitter.com/dai_shi).
