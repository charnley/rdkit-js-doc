---
layout: page
title: RDKit.js Documentation
permalink: /
---

RDKit.js is the official JavaScript distribution of cheminformatics functionality from the [RDKit](https://github.com/rdkit/rdkit) — a C++ library for cheminformatics.

The core WASM module comes from RDKit's [MinimalLib](https://github.com/rdkit/rdkit/tree/master/Code/MinimalLib).
MinimalLib is a C++ layer that wraps a subset of RDKit's API so it can be compiled to WebAssembly and used from JavaScript.
The package is build and published directly from RDKit, while keeping JavaScript documentation here.

The package itself consist only of `.js`, `.wasm`, and `.d.ts` for types, with zero dependencies.
If high-level component javascript is needed, this needs to be implemented yourself and won't be included in the general package.

## Getting Started Using RDKit.js

You can install it using one of the many javascript package managers

```bash
npm i @rdkit/rdkit
yarn add @rdkit/rdkit
pnpm i @rdkit/rdkit
```

Or use the CDN, by adding this script tag to your HTML

```html
<script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
```

## Loading the WASM module

RDKit is a C++ library that has been compiled to WebAssembly (WASM), allowing it to run in browsers and Node.js.
Loading the JavaScript bundle exposes `initRDKitModule()`, which initializes the WASM module and returns an RDKit library object.

- [ ] TODO Note on SINGLE_FILE=1, cannot be used because rdkit is binary is a bit large, better seperate

RDKit runs in its own WASM memory space, separate from JavaScript memory.
The module can be loaded inside a Web Worker to perform computationally intensive operations without blocking the main thread.

WASM loading is asynchronous. `initRDKitModule()` returns a Promise, so you load it with either `.then` chain or `await`.

```js
let GlobalRDKit;

initRDKitModule().then(function (RDKit) {
  console.log("RDKit version: " + RDKit.version());

  // Set RDKit either as a global variable, or in the browser window object
  GlobalRDKit = RDKit;
  window.RDKit = RDKit;
})
```

```js
const GlobalRDKit = await initRDKitModule();
```

When you want to use RDKit with different bundlers or frameworks, some tricks are needed.
See the examples for each framework. Usually this means you can either;

- [ ] TODO Explain the different standard approaches

1. Custom Vite plugin (this repo's approach) — serve/copy .wasm from node_modules. Avoids manual copy, keeps dist/ clean for rebuilds.
1. CDN-hosted — .wasm on external URL, locateFile points there. No build dependency but needs internet.
1. ESM-integrated (newer) — some WASM toolchains emit .js + .wasm as ES module imports. Vite can handle these natively with ?url suffix or top-level await.
1. Copy `.wasm` to a `public/` — .wasm file in public/ or dist/, fetched at runtime. Simplest. No tooling needed.

## Quick Start

When you have initialized the library, you can for example embed a molecule as svg

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var svg = mol.get_svg();
document.getElementById("drawing").innerHTML = svg;
mol.delete(); // always free memory when done
```

See the [demos](/demo)

## License

BSD 3-Clause.
