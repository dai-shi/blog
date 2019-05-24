---
title: "My first WebAssembly trial with Node.js only"
subtitle: "Like LISP"
date: 2019-01-22T12:00:00+09:00
tags: ["wasm", "node"]
---

### Introduction

One of the technologies we should keep eyes on in the web world is WebAssembly or wasm. As I am new to it, I won't describe much of its fundamentals. Please visit the official web site to learn from scratch.

https://webassembly.org/

This short article describes my first trial on wasm. What I want to try is just to run example code, but I'd start with writing the example code in wasm(wat).

### Hand crafted wasm code

**Wasm** is a binary format, but there's a textual representation of it called **wat**. It's written in S-exp. Here's my first example.

```lisp
(module
  (func $fib (param $i i32) (result i32)
    (if (result i32)
      (i32.le_s (get_local $i) (i32.const 1))
      (then (get_local $i))
      (else
        (i32.add
          (call $fib (i32.sub (get_local $i) (i32.const 1)))
          (call $fib (i32.sub (get_local $i) (i32.const 2)))))))
  (export "fib" (func $fib)))
```

Can you guess what this code is? The almost-equivalent code in JavaScript is the following.

```javascript
export const fib = (i) => {
  if (i <= 1) {
    return i;
  } else {
    return fib(i - 1) + fib(i - 2);
  }
};
```

Or:

```javascript
export const fib = i => (i <= 1 ? i : fib(i - 1) + fib(i - 2));
```

### Compiling wat to wasm

There's a tool called wat2wasm in [wabt](https://github.com/webassembly/wabt), which is something you need to build with gcc. Fortunately, I found [wat2wasm](https://www.npmjs.com/package/wat2wasm) in npm, which is probably compiled in wasm. Let's name the example wat code `slow_fib.wat` and do the following.

```bash
$ npx wat2wasm slow_fib.wat
```

That's it. You get `slow_fib.wasm`.

```bash
$ wc slow_fib.*
       1       6      61 slow_fib.wasm
      10      39     329 slow_fib.wat
      11      45     390 total
```

How small it is.

### Running wasm in Node

Let me show how to run the code in Node REPL.

```javascript
$ node
> const fs = require('fs')
undefined
> const buf = fs.readFileSync('./slow_fib.wasm')
undefined
> let res
undefined
> (async () => { res = await WebAssembly.instantiate(buf); })()
Promise
> const { fib } = res.instance.exports
undefined
> fib(10)
55
> [0, 1, 2, 3, 4, 5, 6, 7, 8].map(fib)
[ 0, 1, 1, 2, 3, 5, 8, 13, 21 ]
```

### Comparing the speed

Let's see how long it takes. Simply measure time:

```javascript
> console.time('wasm fib'); fib(40); console.timeEnd('wasm fib');
wasm fib: 716.270ms
undefined
```

Now, how about it in JS?

```javascript
> const fib2 = i => i <= 1 ? i : fib2(i - 1) + fib2(i - 2);
undefined
> console.time('js fib'); fib2(40); console.timeEnd('js fib');
js fib: 1047.255ms
undefined
```

Not that big difference? Maybe, because it is a recursive function. I'm not so sure.

### Summary

This article showed an example to write wasm code in wat, compile it to wasm binary, and run it in Node REPL. Intentionally, the wat code is written like LISP for fun, which might not be a usual compiler output. I'm still learning the wasm [semantics](https://webassembly.org/docs/semantics/), and try something else in the future.
