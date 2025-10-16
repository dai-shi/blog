---
title: "When I Use Valtio and When I Use Jotai"
subtitle: "My two apps use them"
date: 2022-01-30T18:00:00+09:00
tags: ["react", "global-state", "proxy", "valtio", "jotai"]
---

## Introduction

Recently, I often got asked about this question:
Which is recommended, valtio or jotai?

For those who are not familiar with them,
they are two out of many state management libraries that I developed.

<https://github.com/pmndrs/valtio>

<https://github.com/pmndrs/jotai>

Now, from the library perspective, their implementations are very different.
However, from the usage perspective, I understand the confusion.
Both serve similar functionalities and we don't usually use both
in a single app.

It might be valuable if I could explain which to use if
I were to develop some apps.

My answer is I would use valtio for data-centric apps and
jotai for component-centric apps.

Let's dive in.

## Data-centric and component-centric approaches

In the past, I had this tweet, mentioning
"React Centric" and "Data Centric".
React component centric approach is an internal store,
where as data centric approach is an external store.

{{< x user="dai_shi" id="1381252474682535938" >}}

In this article, our focus is the columns in the table.
(The rows are from a different perspective.)

Here's another tweet with the same idea.
It's "state resides at component level (inside react)"
vs.
"state resides at module level (outside react)".

{{< x user="dai_shi" id="1342464894155640833" >}}

Again, our focus is the columns in the table.

So, what are the two approaches?

The data-centric approach is you have data first regardless of
React components.
React components are used to represent those data.
For example, in game development, it's likely that
you may have game state in advance to design components.
You don't want these data to be controlled by React lifecycle.

On the other hand, with the component-centric approach,
you would design components first.
Some states can be locally defined in components with `useState`.
Other states will be shared across components.
For example, in a GUI intensive app, you want to control UI parts
in sync, but they are far away in the component tree.

Note that this is not a ground rule. We could store
game state in the component-centric libraries,
and UI state in the data-centric libraries.
Each library has its own features, so that would be
the point of choice.

I would choose valtio for data-centric apps
and jotai for component-centric apps.

Let's see the actual examples.

## My apps with valtio and jotai

There are two apps I developed each with valtio and jotai.

The first app is called Remote Faces and it uses valtio.
It's an app to share your face image with your colleagues
to show presense in a remote-work environment.

<https://remote-faces.js.org>

It has data to be shared with other users.
The latest version uses [valtio-yjs](https://github.com/dai-shi/valtio-yjs)
to sync data with others.

See the GitHub repo for more details:

<https://github.com/dai-shi/remote-faces>

The second app is called Katachidraw and it uses jotai.
This is an SVG-based drawing app.

<https://katachidraw.vercel.app>

It's actually developed to demonstrate how jotai can
extensively used.

The source code is available:

<https://github.com/dai-shi/katachidraw>

You can also learn the basics in
[this egghead course](https://egghead.io/courses/manage-application-state-with-jotai-atoms-2c3a29f0?af=egwzez).

## Summary

It's really hard to suggest which libraries to use in general.
The real recommendation is to learn both and understand them.

If data-centric approach vs. component-centric approach discussion
makes sense, it may help you to choose one. But still, other features
in valtio and jotai are very different. So, you want to at least
read their README files.

Another suggestion is, if you really like the syntax of valtio,
pick valtio, otherwise pick jotai.
If you are even unsure about it, just pick jotai which has less gotchas.

I didn't discuss other libraries in this article.
It will be more complicated to compare three or more libraries.
Maybe another pair of libraries would be possible to discuss.
