---
layout: page
title: RDKit.js Documentation
permalink: /
---

RDKit.js is the official JavaScript distribution of cheminformatics functionality from the [RDKit](https://github.com/rdkit/rdkit) — a C++ library for cheminformatics.

The project leverages WebAssembly to expose a subset of the RDKit functionality in any JavaScript context. The WASM module bundled with this package is compiled directly from the RDKit source code.

---

## Installation

```bash
npm i @rdkit/rdkit
yarn add @rdkit/rdkit
pnpm i @rdkit/rdkit
# ... etc
```

### Option 1: Copy distribution files

Once installed, copy these two files to your deployed assets at the same location:

- `node_modules/@rdkit/rdkit/dist/RDKit_minimal.js`
- `node_modules/@rdkit/rdkit/dist/RDKit_minimal.wasm`

To load the WASM file from a different path, use the `locateFile` option:

```js
window.initRDKitModule({ locateFile: () => '/path/to/RDKit_minimal.wasm' }).then(/* ... */);
```

### Option 2: Use CDN (unpkg)

```html
<script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
```

---

## Quick Start

Load the JS file and initialize the WASM module in your `<head>`:

```html
<head>
  <script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
  <script>
    window
      .initRDKitModule()
      .then(function (RDKit) {
        console.log("RDKit version: " + RDKit.version());
        window.RDKit = RDKit;
        // RDKit is now ready to use
      })
      .catch(() => {
        // handle loading errors
      });
  </script>
</head>
```

### Draw a molecule

```js
// SVG
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var svg = mol.get_svg();
document.getElementById("drawing").innerHTML = svg;
mol.delete();
```

```js
// HTML5 Canvas
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var canvas = document.getElementById("canvas");
mol.draw_to_canvas(canvas, -1, -1);
mol.delete();
```

### Substructure search with highlighting

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var qmol = window.RDKit.get_qmol("O=C");
var details = mol.get_substruct_match(qmol);
mol.draw_to_canvas_with_highlights(canvas, details);
mol.delete();
qmol.delete();
```

### Drawing options

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var details = {
  atoms: [0, 1, 10],
  explicitMethyl: true,
  addAtomIndices: true,
  legend: 'aspirin'
};
mol.draw_to_canvas_with_highlights(canvas, JSON.stringify(details));
mol.delete();
```

### Coordinate alignment with template

```js
var mol = window.RDKit.get_mol("c1cnc(C)nc1C(=O)O");
var template = window.RDKit.get_mol(`
  Mrv2014 10192005332D
  0  0  0     0  0            999 V3000
M  V30 BEGIN CTAB
M  V30 COUNTS 6 6 0 0 0
M  V30 BEGIN ATOM
M  V30 1 C -5.7917 2.5817 0 0
M  V30 2 N -7.1253 1.8117 0 0
M  V30 3 C -7.1253 0.2716 0 0
M  V30 4 C -5.7917 -0.4984 0 0
M  V30 5 C -4.458 0.2716 0 0
M  V30 6 N -4.458 1.8117 0 0
M  V30 END ATOM
M  V30 BEGIN BOND
M  V30 1 1 1 2
M  V30 2 2 2 3
M  V30 3 1 3 4
M  V30 4 2 4 5
M  V30 5 1 5 6
M  V30 6 2 1 6
M  V30 END BOND
M  V30 END CTAB
M  END
`);
mol.generate_aligned_coords(template, true);
var details = mol.get_substruct_match(template);
mol.draw_to_canvas_with_highlights(canvas, details);
mol.delete();
template.delete();
```

---

## Framework Examples

- [Vanilla JavaScript]({{ site.baseurl }}/examples/vanilla-js)
- [React]({{ site.baseurl }}/examples/react)
- [Vue 3]({{ site.baseurl }}/examples/vue)
- [Angular]({{ site.baseurl }}/examples/angular)
- [Svelte]({{ site.baseurl }}/examples/svelte)
- [Next.js]({{ site.baseurl }}/examples/nextjs)
- [Node.js]({{ site.baseurl }}/examples/node)

---

## Demo

Try the [live demo]({{ site.baseurl }}/demo/) to experiment with SMILES, SMARTS, Canvas rendering, and drawing options.

---

## Source

[github.com/rdkit/rdkit-js](https://github.com/rdkit/rdkit-js)

---

## Contributing

Contributions welcome. See [CONTRIBUTING.md]({{ site.baseurl }}/CONTRIBUTING) for details.

## License

BSD 3-Clause. See [LICENSE]({{ site.baseurl }}/LICENSE).
