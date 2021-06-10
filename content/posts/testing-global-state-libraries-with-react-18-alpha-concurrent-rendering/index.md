---
title: "Testing Global State Libraries With React 18 Alpha Concurrent Rendering"
subtitle: "What are tearing and branching"
date: 2021-06-10T21:00:00+09:00
tags: ["react", "hooks", "global-state", "redux", "context", "concurrent"]
---

## Introduction

The React team announced [The Plan for React 18](https://reactjs.org/blog/2021/06/08/the-plan-for-react-18.html) with various features.
Among them, concurrent rendering is one of my interests
because I have been developing various global state libraries
and there are known issues called "tearing" and "branching".

Tearing is an undesirable behavior in which
two components render with different values for same global state.

Branching is a desirable behavior in which
a component renders with a stale value for a state in a transition update.

Both are subtle behaviors and
would not be required in most moderate apps.

Nevertheless, out of curiosity, I made a small experimental tool
to test the behavior of various libraries in concurrent rendering.

## Tool

It's called "Will this React global state work in concurrent rendering?"

<https://github.com/dai-shi/will-this-react-global-state-work-in-concurrent-rendering>

It runs a tiny counter app for each library and test behaviors.

There are 6 tests.

- with useTransition
  - test 1: updated properly with transition
  - test 2: no tearing with transition
  - test 3: ability to interrupt render
  - test 4: proper branching with transition
- with intensive auto increment (EXPERIMENTAL)
  - test 5: updated properly with auto increment
  - test 6: no tearing with auto increment

Test 5 and test 6 are something tried to distinguish
useMutableSource behavior, but it doesn't work totally as expected yet.

## Results

![screenshot1](./screenshot1.png "Screenshot of result table")

How do we read this table?

Test 1 is to see if all counts are updated properly.
Three libraries fail with this. 
I assume they are somewhat batching updates too aggressively.

Test 2 is one of the goals. There are two libraries fail,
but as you see redux with useMutableSource passes.
I expect any library with useMutableSource would pass this.

Test 3 is to see if time slicing is effective.
Failing this means, it doesn't get the benefit of concurrent rendering.

Test 4 is another goal. Branching would only work with
React state and most global state libraries use module state,
failing this test is rather expected.

Test 5 and test 6 are experimental. They show some differences.
What can be said at this moment is React state passes both tests.

## Closing notes

This was a short post to present my recent experiment with React 18 alpha.
I thought it's better than a tweet to explain how I see the results.
Please note that they are just the results as of writing and
they may not be accurate. Tests can be wrong.
If you are interested, please clone the repository and try it by yourself.

