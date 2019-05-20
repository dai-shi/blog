---
title: "Clean Expo React Native React Apollo Graphql Typescript Boilerplate"
date: 2018-10-20T12:00:00+09:00
tags: ["expo", "react-native", "apollo", "graphql", "typescript"]
---

### Motivation

Have you heard of React Native, GraphQL and TypeScript? I'm looking for a easy and nice tech stack to develop mobile apps. These three may fit nicely for developing mobile apps for beginners up to intermediates. More precisely, Expo is chosen for building React Native development environment, and React Apollo is for GraphQL. These libraries comes with type annotations, which allow us to code in TypeScript. By "Clean" I mean not only the boilerplate is minimalistic, but also we don't allow "any" types.

### Features

This boilerplate consists of:

- Expo
- React Apollo (+ Apollo Client)
- TypeScript (+ tslint)

The points of this boilerplate are the followings:

- Only basic packages are included
- No "any" types including implicit ones are allowed
- GraphQL is mocked in the client side, so no server is required to run the app in the first place.

### The repository

https://github.com/dai-shi/typescript-expo-apollo-boilerplate

### Snippets

Please checkout the code in the repository, but here's a piece of the code just for a reference.

```jsx
const NewPost = () => (
  <Mutation mutation={ADD_POST} refetchQueries={['queryPosts']}>
    {(addPost) => {
      const add = (text: string) => {
        addPost({ variables: { text } });
      };
      return (
        <MyTextInput onSubmit={add} />
      );
    }}
  </Mutation>
);
```

### Notes

The code will probably be improved. For example, the tslint config might not be enough. Also, I usually prefer SFCs to classes, and I found that recompose helps for it because it has type annotations.
