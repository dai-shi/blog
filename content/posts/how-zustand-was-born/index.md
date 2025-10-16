---
title: "How Zustand Was Born"
subtitle: "More precisely, since Zustand v3"
date: 2024-07-14T13:00:00+09:00
tags: ["react", "global-state", "typescript", "zustand"]
---

### Introduction

In this post, I would like to share the story behind Zustand's development. To be precise, I wasn't the original author of Zustand, and when Zustand v0 was born, I was developing other global state libraries, especially React-Tracked. By the way, I now consider myself a (secondary) author of Zustand.

I found my tweet mentioning Zustand, comparing it with other libraries, including mine.

{{< x user="dai_shi" id="1141004414129324032" >}}

My belief at that point was that the global state should be passed via React Context so that it works with React Concurrent Mode. So, I made a comparison table to differentiate my library and others, and Zustand was one of them. This was in the year 2019.

### Zustand v3

In 2020, I joined the Poimandres group and took over Zustand's development. My interest back then was to make global state libraries work with React Concurrent Mode. It wasn't possible to get the full benefit from Concurrent Mode, but there was an experimental API called `useMutableSource` to make the global state compatible with Concurrent Mode.

I was experimenting with many things using a React Context-based solution with React-Tracked and wondered what we could do with the global state without React Context. Zustand was one year old, but nobody was maintaining it. So, I decided to take it over.

The experimental `useMutableSource` API wasn't ready, so the first task was to update various things and fix some bugs. This was when Zustand v3 was born. My hope was to soon release v4 with `useMutableSource`, but it didn't happen. There's another story behind it.

Nowadays, it's a well-known pattern to have the global state outside and optionally use React Context to pass its store. Zustand was a pioneer in this pattern. People were very skeptical about not having the global state in React Context, and we had a hard time explaining that it's a valid pattern.

### Current Status

One of the things I care about with Zustand is its simple implementation and its small bundle size. If you look at the source code, you'll see it's nothing more than just using React hooks with a minimal store implementation.

As of writing, Zustand v4 is the latest version, which has very advanced TypeScript support, and the code is almost entirely rewritten from v3. We have Zustand v5 almost ready for the next release.

Last but not least, there are several contributors who maintain this project. I didn't expect this would happen when I took over the project. I'm very grateful for it. Thanks, everyone.
