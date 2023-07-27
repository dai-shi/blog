---
title: "How to Use Jotai and useTransition for Mutation"
subtitle: "The power of promise values"
date: 2023-07-28T01:00:00+09:00
tags: ["react", "global-state", "jotai", "suspense"]
---

### Introduction

[Jotai](https://github.com/pmndrs/jotai)
is a powerful library designed for seamless integration with React Suspense.
Unlike Zustand, Jotai's primary focus is React Suspense from the beginning.

In its simplest form, Jotai effortlessly works with Suspense,
and gives superior developer experience and user experience with async data.

<https://codesandbox.io/s/vjfpcd>

```tsx
import { Suspense } from "react";
import { atom, useAtomValue, useSetAtom } from "jotai";

const id = 1;

const postAtom = atom(async (get) => {
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
  const data = await res.json();
  return data;
});

const Post = () => {
  const { title } = useAtomValue(postAtom);
  return <div>Title: {title}</div>;
};

const App = () => (
  <Suspense fallback="Loading...">
    <div>
      <Post />
    </div>
  </Suspense>
);

export default App;
```

There are no issues so far.

This blog post explores Jotai's strength in handling mutations
using promise values in atoms and useTransition.
Let's learn how to leverage React Suspense with Jotai to handle async data.

### The problem

Now, if we want to mutate the server data,
our first option would be a derived atom with an async write function.
For simplicity, let's use a write-only atom.

<https://codesandbox.io/s/z84n9x>

```tsx
import { Suspense, useTransition } from "react";
import { atom, useAtomValue, useSetAtom } from "jotai";

const id = 1;

const mutationResultAtom = atom<{ title: string } | null>(null);

const postAtom = atom(async (get) => {
  const mutationResult = get(mutationResultAtom);
  if (mutationResult) {
    return mutationResult;
  }
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
  const data = await res.json();
  return data;
});

const mutateAtom = atom(null, async (get, set, title) => {
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ title })
  });
  const data = await res.json();
  set(mutationResultAtom, data);
});

const Post = () => {
  const { title } = useAtomValue(postAtom);
  return <div>Title: {title}</div>;
};

const Mutate = () => {
  const mutate = useSetAtom(mutateAtom);
  const handleClick = () => {
    mutate("changed title");
  };
  return (
    <div>
      <button onClick={handleClick}>Click me</button>
    </div>
  );
};

const App = () => (
  <Suspense fallback="Loading...">
    <div>
      <Post />
      <Mutate />
    </div>
  </Suspense>
);

export default App;
```

Currently, it doesn't suspend during mutation,
which causes it to not work well with useTransition.

What we would like to do is the following:

```tsx
  const [isPending, startTransition] = useTransition();
  const handleClick = () => {
    startTransition(() => {
      mutate("changed title");
    });
  };
```

However, it's a NO-OP because we resolve a promise
before updating the post atom.
We need to avoid resolving a promise in the write function,
and resolve the promise in the read function to make an atom to suspend.

### The solution

We want to tell React to suspend on mutation. How can we achieve that?

The solution is to store a promise value in an atom,
and resolve it in the read function.

Let's see what it looks like.

<https://codesandbox.io/s/fkjvrg>

```tsx
import { Suspense, useTransition } from "react";
import { atom, useAtomValue, useSetAtom } from "jotai";

const id = 1;

const mutationResultAtom = atom<Promise<{ title: string }> | null>(null);

const postAtom = atom(async (get) => {
  const mutationResult = await get(mutationResultAtom);
  if (mutationResult) {
    return mutationResult;
  }
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
  const data = await res.json();
  return data;
});

const mutateAtom = atom(null, (get, set, title) => {
  const data = fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ title })
  }).then((res) => res.json());
  set(mutationResultAtom, data);
});

const Post = () => {
  const { title } = useAtomValue(postAtom);
  return <div>Title: {title}</div>;
};

const Mutate = () => {
  const mutate = useSetAtom(mutateAtom);
  const [isPending, startTransition] = useTransition();
  const handleClick = () => {
    startTransition(() => {
      mutate("changed title");
    });
  };
  return (
    <div>
      <button onClick={handleClick}>Click me</button>
      {isPending && "Pending..."}
    </div>
  );
};

const App = () => (
  <Suspense fallback="Loading...">
    <div>
      <Post />
      <Mutate />
    </div>
  </Suspense>
);

export default App;
```

Now, it works with useTransition,
and it shows "Pending..." while performing mutations.

This pattern also works well when a mutation doesn't return updated data.
In such cases, we can discard the mutation result and fetch the data again.

### A more complex case

In many cases, an async atom depends on another atom.
In our example, let's assume the id is in an atom `idAtom`.

We need a small trick, but this is how it works.

<https://codesandbox.io/s/jpfqwy>

```tsx
import { Suspense, useTransition } from "react";
import { atom, useAtomValue, useSetAtom } from "jotai";

const idAtom = atom(1);

const mutationResultAtom = atom<{
  id: number;
  data: Promise<{ title: string }>;
} | null>(null);

const postAtom = atom(async (get) => {
  const id = get(idAtom);
  const mutationResult = get(mutationResultAtom);
  if (mutationResult && mutationResult.id === id) {
    return mutationResult.data;
  }
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
  const data = await res.json();
  return data;
});

const mutateAtom = atom(null, (get, set, title) => {
  const id = get(idAtom);
  const data = fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ title })
  }).then((res) => res.json());
  set(mutationResultAtom, { id, data });
});

const Post = () => {
  const { title } = useAtomValue(postAtom);
  return <div>Title: {title}</div>;
};

const Next = () => {
  const setId = useSetAtom(idAtom);
  const [isPending, startTransition] = useTransition();
  const handleClick = () => {
    startTransition(() => {
      setId((id) => id + 1);
    });
  };
  return (
    <div>
      <button onClick={handleClick}>Next Post</button>
      {isPending && "Pending..."}
    </div>
  );
};

const Mutate = () => {
  const mutate = useSetAtom(mutateAtom);
  const [isPending, startTransition] = useTransition();
  const handleClick = () => {
    startTransition(() => {
      mutate("changed title");
    });
  };
  return (
    <div>
      <button onClick={handleClick}>Change title</button>
      {isPending && "Pending..."}
    </div>
  );
};

const App = () => (
  <Suspense fallback="Loading...">
    <div>
      <Post />
      <Next />
      <Mutate />
    </div>
  </Suspense>
);

export default App;
```

The trick is to associate the mutation result with the corresponding id.
This technique can be applied to handle more complex scenarios as well.

### Closing notes

Jotai's seamless integration with useTransition
also works for handling mutations.
By leveraging promise values, we can develop better user experience,
even with complex state models.

Hope you find it useful and discover the true power of Jotai.
