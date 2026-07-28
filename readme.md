---
layout: page
title: RDKit.js Documentation
permalink: /
---

[![NPM Latest Version](https://img.shields.io/npm/v/@rdkit/rdkit)](https://www.npmjs.com/package/@rdkit/rdkit)
[![NPM Weekly Downloads](https://img.shields.io/npm/dw/@rdkit/rdkit)](https://www.npmjs.com/package/@rdkit/rdkit)
[![NPM Monthly Downloads](https://img.shields.io/npm/dm/@rdkit/rdkit)](https://www.npmjs.com/package/@rdkit/rdkit)
[![NPM Yearly Downloads](https://img.shields.io/npm/dy/@rdkit/rdkit)](https://www.npmjs.com/package/@rdkit/rdkit)
[![NPM Total Downloads](https://img.shields.io/npm/dt/@rdkit/rdkit?label=total%20downloads)](https://www.npmjs.com/package/@rdkit/rdkit)

RDKit.js is the official JavaScript distribution of cheminformatics functionality from the [RDKit](https://github.com/rdkit/rdkit) - a C++ library for cheminformatics.

The core WASM module comes from RDKit's [MinimalLib](https://github.com/rdkit/rdkit/tree/master/Code/MinimalLib).
MinimalLib is a C++ layer that wraps a subset of RDKit's API so it can be compiled to WebAssembly and used from JavaScript.
The package is build and published directly from RDKit, while keeping JavaScript documentation here.

The package itself consist only of three files;

- `RDKit_minimal.js` - Standard JavaScript wrapper for loading WASM modules
- `RDKit_minimal.wasm` - The compiled RDKit MinimalLib WASM binary
- `RDKit_minimal.d.ts` - TypeScript interface generated during compilation.

That means the package has zero dependencies and if high-level component javascript is needed, this needs to be implemented yourself and won't be included in the general package.
This is to ensure easy maintenance of the package.

## Install RDKit JS

You can install it using one of the many (and growing) list of javascript package managers

```bash
npm i @rdkit/rdkit
yarn add @rdkit/rdkit
pnpm i @rdkit/rdkit
...
```

Or use a CDN, by adding this script tag to your HTML.

```html
<script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
```

## Loading the WASM module

The JavaScript wrapper exposes `initRDKitModule()`, which initializes the WASM module and returns an RDKit library object.

RDKit runs in its own WASM memory space, separate from JavaScript memory.
The module can be loaded inside a Web Worker to perform computationally intensive operations without blocking the main thread.

WASM loading is asynchronous.
`initRDKitModule()` returns a Promise, so you load it with either `.then` chain or `await`.

```js
let GlobalRDKit;

initRDKitModule().then(function (RDKit) {
  console.log("RDKit version: " + RDKit.version());

  // Set RDKit either as a global variable, or in the browser window object
  // Note, some frameworks will not like the window-approach because that
  // requires a browser runtime. For CDN approach, you will need the
  // window-approach though.
  GlobalRDKit = RDKit;
  window.RDKit = RDKit;
})

// Or using await
const GlobalRDKit = await initRDKitModule();
```

When you want to use RDKit with different bundlers or frameworks, some tricks are needed.
Usually a trick is needed to make the `.wasm` file available as a standalone file, but it really varies.
Also, because of the output `.js` file does some node checks, the bundlers have problems with node (not javascript) specific functions.
We have created examples for Vanilla JS, React, Vue, Angular, Svelte, Next.js and Node.js.
However, these are not RDKit specifc hacks, but generically how you setup WASM support for those frameworks.

- [ ] TODO Explain the different generic approaches

1. Custom Vite plugin (this repo's approach) — serve/copy .wasm from node_modules. Avoids manual copy, keeps dist/ clean for rebuilds.
1. CDN-hosted — .wasm on external URL, locateFile points there. No build dependency but needs internet.
1. ESM-integrated (newer) — some WASM toolchains emit .js + .wasm as ES module imports. Vite can handle these natively with ?url suffix or top-level await.
1. Copy `.wasm` to a `public/` — .wasm file in public/ or dist/, fetched at runtime. Simplest. No tooling needed.


This branch is dead in the browser (the `if` is false at runtime). But
esbuild parses the file at build time and tries to resolve every `import`
it sees, including `"node:module"`. The build fails because `node:` built-ins
do not exist in a browser bundle.

## Getting started

When you have initialized the library, you can for example embed a molecule as svg

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var svg = mol.get_svg();
document.getElementById("drawing").innerHTML = svg;
mol.delete(); // always free memory when done
```

`mol.delete()` frees the WASM-side object, as emscripten allocations are not garbage-collected.

## License

The binary is compiled directly from RDKit, so license is unchanged.

BSD 3-Clause.

## Citation

See [rdkit.com](https://rdkit.com) for citation.
Note the version installed.
