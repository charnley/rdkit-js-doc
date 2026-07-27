---
layout: page
title: Using RDKit.js with Node.js
menu: Node.js
permalink: /examples/node/
---

Minimal example showing how to run RDKit WebAssembly build in a Node.js environment, e.i. JavaScript backend without a browser.
Using `Node.js >= 18`, 

First initialize `package.json` with ESM, by setting `"type": "module"` in `package.json`.
For example a `package.json`:

```json
{
  "name": "rdkit-minimal-nodejs-example",
  "version": "1.0.0",
  "description": "Minimal Node.js example using RDKit WASM",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "keywords": ["rdkit", "wasm", "nodejs", "chemistry"],
  "license": "BSD-3-Clause"
}
```

The `"type": "module"` line is required — without it, `import` will fail, or give a warning like;

    Reparsing as ES module because module syntax was detected.

Then install rdkit

```bash
npm install @rdkit/rdkit
```

Which can be accessed by `index.js`:

```js
// ./index.js
import initRDKit from "@rdkit/rdkit";
const rdkit = await initRDKit();

console.log("RDKit version:", rdkit.version());

const SMILES = "CCO";
const mol = rdkit.get_mol(SMILES);
if (!mol) {
  console.error("Failed to parse SMILES:", SMILES);
  process.exit(1);
}

const svg = mol.get_svg(300, 300);
console.log("SVG length:", svg.length);

mol.delete();
```
