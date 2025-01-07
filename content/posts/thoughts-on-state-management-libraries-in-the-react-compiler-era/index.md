---
title: "Thoughts on State Management Libraries in the React Compiler Era"
subtitle: "RSC also matters"
date: 2025-01-07T22:00:00+09:00
tags: ["react", "global-state", "compiler", "rsc", "zustand", "jotai", "valtio"]
---

## Introduction

I've been developing three React state management libraries: Zustand, Jotai, and Valtio. So, how are they going to hold up in the new era with the React Compiler and React Server Components (RSCs)? This post shares my random thoughts on the topic.

## What are state management libraries?

State management libraries manage the state of your application. For my libraries specifically, the primary goal isn't just about how the state is managed. This ties into why there are three libraries. They differ in how they deal with states (for example, immutable, atomic, proxy, etc.) and how developers manage their states.

The primary goal of what I call global state libraries is to overcome the limitations of React's state solutions: useState and useContext. With React Context, it's difficult to avoid unnecessary re-renders because React triggers re-renders even when only a small part of the state changes. Global state libraries address this by providing their own mechanisms to subscribe to state changes and trigger re-renders efficiently.

## How does the React Compiler change the game?

The React Compiler transforms React user code, producing optimized code with full memoization. In short, we no longer need to manually memoize with `useMemo`, `useCallback`, or `React.memo`. So, why does this matter to global state libraries?

Essentially, the React Compiler will address the limitations of React Context. As of this writing, it doesn't optimize re-renders yet, but if it becomes advanced enough, React will be able to avoid unnecessary re-renders with React Context. The core problem that global state libraries are designed to solve may disappear.

## How do React Server Components come into play?

React Server Components (RSCs) allow React components to be rendered on the server, sending serialized results over the wire. While this is a powerful feature, does it impact global state libraries?

Compared to the React Compiler, RSCs have a less direct effect. However, some global state libraries are used for caching server state. While developers might already use other libraries for this purpose, if global state libraries handle server state caching, RSCs could reduce their necessity. What remains for global state libraries could be minimal, and they might not need to "manage" complex states anymore.

## What should global state libraries do to survive?

As I mentioned, the primary goal of global state libraries is to avoid unnecessary re-renders. However, their value extends beyond that. These libraries offer ways to organize states, compose states, and connect to external services. Those capabilities remain useful even if the primary goal becomes obsolete.

Moreover, enabling client-side state to work seamlessly with server-side state would be a powerful feature. It could simplify server interactions and improve developer experience. While I don't have a concrete idea yet, this is one of the reasons I started developing [Waku](https://waku.gg).

Now, let's take a brief look at each library."

### Zustand

Aside from render optimization with selectors, the biggest benefit of Zustand is that its state exists outside of React. This is something React state with Context cannot address, so Zustand will continue to have a role.

The biggest risk for Zustand is its simplicity. In the Compiler era, a new library offering a more straightforward solution could replace it. However, its simplicity is also its strength. It can be maintained by anyone and thus will remain viable in the long term.

### Jotai

Jotai was initially designed to avoid extra re-renders, but it introduced a new principle: atoms, popularized by Recoil. Atoms allow defining state outside of React while keeping it managed within React. Jotai also offers an option to create a store that holds atom values outside of React. The strength of the atom model lies in its declarative nature and composability, enabling the creation of a data flow graph for managing complex state.

The biggest risk for Jotai is that many users may not need to manage such complex state. For simpler use cases, React state with Context might be enough. However, the declarative atom model could be expanded to the server side, which might be a promising direction.

### Valtio

Valtio has two main aspects. First, it bridges the gap between mutable and immutable state. JavaScript objects are inherently mutable, and `valtio/vanilla` provides a way to derive immutable snapshots from mutable state. Second, `valtio/react` offers render optimization via proxies, which works magically behind the scenes.

The render optimization in Valtio conflicts conceptually with the React Compiler. It's a runtime solution vs. a build-time solution. That said, Valtio v2 works perfectly with the React Compiler. While the second aspect may lose value in the Compiler era, the first aspect, managing state with plain JavaScript objects, remains highly useful. Few other libraries offer this capability.

## Closing Notes

While I feel some concerns about the future of global state libraries, nothing will change anytime soon. The React Compiler is still experimental, and RSC is still new. It will take time for people to adapt to this new era. In the meantime, global state libraries will continue to play a role in the React ecosystem.

As I discussed, the three libraries still hold value even in the new era. However, I believe I should take action to ensure they can survive in the future. After all, if things don't evolve, they won't last.
