---
title: "How I Got Involved in OSS"
subtitle: "My Journey to Developing React libraries"
date: 2024-09-03T22:00:00+09:00
tags: ["react", "oss"]
---

### Introduction

In this post, I would like to reflect on my journey in open source software development.

### My first OSS contribution

Around the year 2000, I made my first contribution to OSS: an Ethernet driver for NetBSD/mac68k. It was written in C, and now, looking back, I can barely understand the code.

Found this link on the Web:

<http://cvsweb.netbsd.org/bsdweb.cgi/src/sys/arch/mac68k/nubus/if_netdock_nubus.c?rev=1.35&content-type=text/x-cvsweb-markup>

### Chicken Scheme

Another phase in my OSS journey was working with Scheme. Although my contributions were small, I did contribute to a few libraries.


Here's one of them:

<https://wiki.call-cc.org/eggref/5/crc>

### JavaScript

In 2012, my first JavaScript library was open-sourced. I have been using GitHub since then.

Link:

<https://github.com/dai-shi/continuation.js>

It was a library that aimed to bring some Scheme flavor to JavaScript, although I eventually had to give up on fully implementing all the features I wanted.

After that, I developed several libraries for Node.js, especially for Express.js.

For example:

<https://github.com/dai-shi/social-cms-backend>

During this period, I sent patches to some OSS projects like JSDOM.

Although my primary focus was on libraries, I also developed some small apps.

For example:

<https://github.com/dai-shi/codeonmobile>

This one was built with Angular.js.

### Meteor

In 2015, I shifted my focus to Meteor.

For example:

<https://github.com/dai-shi/meteor-blaze-google-maps>

It was around this time that I switched to using MIT license, having previously used BSD license.

### React

In 2016, I started developing React libraries.

I believe this was the first one:

<https://github.com/dai-shi/react-compose-state>

This was likely the start of my journey to global state management. I wanted to use state with function components. But, Redux was too much for some (most?) cases. Nobody used this library. People used recompose.

There are more libraries I developed, but unfortunately none of them got much attention.

### React Hooks

In 2018, React Hooks was announced. I felt it was the perfect time to develop something based on this new concept. I took this opportunity seriously, and started to put most of my time on it.

One of my new activities was writing blog posts. It was good to get attention and feedback.

My interest in React Hooks was mainly in two fields:
- Global state management with concurrent mode in mind
- Suspense for data fetching

I developed numerous libraries during this period, some of which are still actively maintained and used in production.

### Poimandres

In 2020, I joined the Poimandres developer collective, where I started developing Zustand, Jotai, and Valtio.

Read more in my other blog posts:
- <https://blog.axlight.com/posts/how-zustand-was-born/>
- <https://blog.axlight.com/posts/how-jotai-was-born/>
- <https://blog.axlight.com/posts/how-valtio-was-born/>

### Current Status

Currently, I'm basically a full-time open source developer. However, I still need to do freelance work to pay the bills, as my sponsorships aren't enough yet. Earning enough to focus solely on open source is my next goal.

Another goal of mine is to evolve the ecosystem with the community. I can maintain the small parts, for example the Jotai core, but I need collaborators to maintain the whole ecosystem, such as Jotai integration libraries. Organizing the community is a big challenge for me.
