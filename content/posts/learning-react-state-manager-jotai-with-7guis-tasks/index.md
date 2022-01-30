---
title: "Learning React State Manager Jotai With 7GUIS Tasks"
subtitle: "Challenge yourself with the tasks"
date: 2021-09-13T20:23:30+09:00
tags: ["react", "hooks", "global-state", "atom", "typescript", "jotai"]
---

## Introduction

I stumbled upon [7GUIS](https://eugenkiss.github.io/7guis/) tasks while reading
[XState Tutorials](https://xstate.js.org/docs/tutorials/7guis/counter.html).
This motivated me to challenge those 7 tasks with
[jotai](https://github.com/pmndrs/jotai).

It turned out that this would be a good resource to learn jotai.
They are from basic tasks to advanced tasked, and you will
see how they are implemented, occasionally magically.

It's recommended to try it yourself first.
If you succeed to implement it, then you can compare.
Even if you fail, you can read and learn.

## Task 1: Counter


> The task is to build a frame containing a label or read-only textfield T and a button B. ...

[See full task description](https://eugenkiss.github.io/7guis/tasks#counter)

This is pretty easy. Good to try for the first time.

Check the codesandbox link in the following tweet.

{{< tweet 1432313981176205319 >}}

## Task 2: Temperature Converter

> The task is to build a frame containing two textfields TC and TF representing the temperature in Celsius and Fahrenheit, respectively. ...

[See full task description](https://eugenkiss.github.io/7guis/tasks#temp)

This is a bit confusing (at least for me),
because converting tempertures seems a best fit for derived atoms.
We need to handle non numeric input and thus it's rather straightforward.

Check the codesandbox link in the following tweet.

{{< tweet 1432325078419587079 >}}

## Task 3: Flight Booker

> The task is to build a frame containing a combobox C with the two options "one-way flight" and "return flight", two textfields T1 and T2 representing the start and return date, respectively, and a button B for submitting the selected flight. ...

[See full task description](https://eugenkiss.github.io/7guis/tasks#flight)

I thought this is faily easy except for
parsing a string for a date.
You would need to keep both string and date for comparison.

Check the codesandbox link in the following tweet.

{{< tweet 1432355221020168196 >}}

## Task 4: Timer

> The task is to build a frame containing a gauge G for the elapsed time e, a label which shows the elapsed time as a numerical value, a slider S by which the duration d of the timer can be adjusted while the timer is running and a reset button R. ...

[See full task description](https://eugenkiss.github.io/7guis/tasks#timer)

Getting hard. We need to take care of timing.
I'm not 100% sure if my implementation is readable enough.

Check the codesandbox link in the following tweet.

{{< tweet 1432723166086971401 >}}

## Task 5: CRUD

> The task is to build a frame containing the following elements: a textfield Tprefix, a pair of textfields Tname and Tsurname, a listbox L, buttons BC, BU and BD and the three labels as seen in the screenshot. ...

[See full task description](https://eugenkiss.github.io/7guis/tasks#crud)

This would be a good challenge to handle a list and filter it.
My implementation uses a technique called atoms-in-atom,
but you could implement without it.

Check the codesandbox link in the following tweet.

{{< tweet 1433116078540935170 >}}

## Task 6: Circle Drawer

> The task is to build a frame containing an undo and redo button as well as a canvas area underneath. Left-clicking inside an empty area inside the canvas will create an unfilled circle with a fixed diameter whose center is the left-clicked point. ...

[See full task description](https://eugenkiss.github.io/7guis/tasks#circle)

This is a fun task. In Web, we can use SVG, so drawing part is trivial.
On the other hand, movable dialog is hard.
I did it in a naive way. There should be some better ways.
Using canvas instead of SVG and using browser window using postMessage
would be advanced challenges.

Check the codesandbox link in the following tweet.

{{< tweet 1433480261547663377 >}}

## Task 7: Cells

> The task is to create a simple but usable spreadsheet application. The spreadsheet should be scrollable. The rows should be numbered from 0 to 99 and the columns from A to Z. Double-clicking a cell C lets the user change C's formula. After having finished editing the formula is parsed and evaluated and its updated value is shown in C. ...

[See full task description](https://eugenkiss.github.io/7guis/tasks#cells)

I wanted to try this task from the start.
I thought this would be very interesting with jotai,
which already has dependency tracking.
The result is it is very interesting.
The code is surprisingly small.
Note that I cheated formula evaluation with `eval`.

Check the codesandbox link in the following tweet.

{{< tweet 1433804219828490241 >}}

## Summary

How was it?
Would you like to challenge yourself?
Even if it's too hard, I suppose reading the implementation
will help you to learn.
I am so impressed that these 7 tasks are well designed.

Enjoy coding.
