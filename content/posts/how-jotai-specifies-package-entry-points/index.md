---
title: "How Jotai Specifies Package Entry Points"
subtitle: "Support CJS and ESM as much as possible"
date: 2022-04-17T17:00:00+09:00
tags: ["node", "react", "jotai"]
---

## Introduction

If someone has already looked into `package.json` in the jotai library,
they may find `"exports"` field.

<https://github.com/pmndrs/jotai/blob/v1.6.4/package.json#L18-L31>

```json
  "exports": {
    "./package.json": "./package.json",
    ".": {
      "types": "./index.d.ts",
      "module": "./esm/index.js",
      "import": "./esm/index.mjs",
      "default": "./index.js"
    },
    "./utils": {
      "types": "./utils.d.ts",
      "module": "./esm/utils.js",
      "import": "./esm/utils.mjs",
      "default": "./utils.js"
    },
```

In the Node.js docs, it is described as
[Package entry points](https://nodejs.org/api/packages.html#package-entry-points).
Node.js v12.7.0 [started implementing it.](https://nodejs.org/en/blog/release/v12.7.0/)
Nowadays, it's used also by bundlers like webpack and vite.

We use package entry points for separating modules.
For example, `jotai` is a core module exporting core functions.
`jotai/utils` is a separate module which exports additional functions
based on core functions.
(By the way, another option is to publish two packages instead.
But, we prefer multiple entry points in a single package.)

This article describes about how the entry point work.
It's based on our observation and might not be 100% accurate.

## Fallback structure

First of all, for tools that don't understand `"exports"`,
we place CJS files traditionally.

```
./index.js
./utils.js
```

This will support file based resolution.

- `require('jotai')` points to `./index.js`
- `require('jotai/utils')` points to `./utils.js`

Old Node.js works with this and maybe old bundlers do too.

## "exports" with default

With "exports", we can export subpath entry points along with main entry point.
We want to support both CJS and ESM,
and [conditional exports](https://nodejs.org/api/packages.html#conditional-exports) should do.
Conditional exports accepts `"default"` as last element for fallback.
We use CJS for fallback because that's the default if `"type"` in package.json is omitted.

This will bring to the following config at minimum:

```json
  "exports": {
    ".": {
      "default": "./index.js"
    },
    "./utils": {
      "default": "./utils.js"
    },
```

We have other subpaths than "utils".
For example, adding "devtools" becomes like this:

```json
  "exports": {
    ".": {
      "default": "./index.js"
    },
    "./utils": {
      "default": "./utils.js"
    },
    "./devtools": {
      "default": "./devtools.js"
    },
```

Note that if [subpath patterns](https://nodejs.org/api/packages.html#subpath-patterns) is supported, we can do this:

```json
  "exports": {
    ".": {
      "default": "./index.js"
    },
    "./*": {
      "default": "./*.js"
    },
```

But subpash patterns are only supported since Node.js v12.20.0.

## Entry point for package.json

If some tools are very strict with `"exports"` and
if we don't have an entry for package.json, they complain.

Hence, we add such an entry:

```json
  "exports": {
    "./package.json": "./package.json",
```

## ESM for Node.js

`"import"` condition is for `import` statement and `import()` expression.
This is a little bit tricky,
but we ended up using `.mjs` extension for this entry.
This indicates ESM to Node.js regardless of `"type"` field in package.json.

As a result, it looks like this:

```json
  "exports": {
    "./package.json": "./package.json",
    ".": {
      "import": "./esm/index.mjs",
      "default": "./index.js"
    },
    "./utils": {
      "import": "./esm/utils.mjs",
      "default": "./utils.js"
    },
```

We chose `./esm` subfolder to place ESM files for some reasons.
But it turns out that it is no longer important
because currently our fallback is CJS.

## "module" for non-Node.js bundlers

Some bundlers don't like the `.mjs` extension
probably because it's not widely used yet.

As far as I know, webpack v5 and vite support
unofficial "module" condition.

So, we can specify it with `.js` extension.

```json
  "exports": {
    "./package.json": "./package.json",
    ".": {
      "module": "./esm/index.js",
      "import": "./esm/index.mjs",
      "default": "./index.js"
    },
    "./utils": {
      "module": "./esm/utils.js",
      "import": "./esm/utils.mjs",
      "default": "./utils.js"
    },
```

Currently, `./esm/index.js` and `./esm/index.mjs` have same content.
If for some reason they couldn't be the same, we would be able to change them.

## How to deal with TypeScript

As far as I understand, tsc looks for the same file name with `.d.ts` extension.
We place type definition files along with JS files.

```
./index.js
./index.d.ts
./utils.js
./index.d.ts
./esm/index.js
./esm/index.mjs
./esm/index.d.ts
./esm/utils.js
./esm/utils.mjs
./esm/utils.d.ts
```

TypeScript will support `"types"` condition in [4.7](https://devblogs.microsoft.com/typescript/announcing-typescript-4-7-beta/#package-json-exports-imports-and-self-referencing).

Having those would be nice:

```json
  "exports": {
    "./package.json": "./package.json",
    ".": {
      "types": "./index.d.ts",
      "module": "./esm/index.js",
      "import": "./esm/index.mjs",
      "default": "./index.js"
    },
    "./utils": {
      "types": "./utils.d.ts",
      "module": "./esm/utils.js",
      "import": "./esm/utils.mjs",
      "default": "./utils.js"
    },
```

We have already added them before knowing TypeScript 4.7 would support it.

Technically, we will be able to delete `./esm/*.d.ts` files with TypeScript 4.7.
We will keep them for older TypeScript versions for the mean time.

## Dual package hazard

[Dual package hazard](https://nodejs.org/api/packages.html#dual-package-hazard)
is a big problem when supporting both ESM and CJS.
Jotai uses some module level variables, so it can suffer from this problem.

So far, we believe the probability of this case is fairly low,
and it's easy to notice if it exists.
We leave this problem unsolved and wait for more feedback.

## Closing note

We described some parts of our `"exports"` entry points
in an unorganized way.
I hoped to have more useful info about how we came there,
but it turns out that it's not very straightforward.
For example, some of old decisions were made with our misunderstanding,
and they are no longer important.
So, this article is just a note about *how we think* at this point.
It's likely that it may not be valid or best in the near future.

Mixing CJS and ESM is really hard, and we hope ecosystem migrates
soon and finds a good pattern meanwhile.

If you want to learn the concrete example,
visit unpkg or something to see the package content:
<https://unpkg.com/browse/jotai@1.6.4/>
