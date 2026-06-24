---
layout: page
title: Node.js
permalink: /examples/node/
---

## Using RDKit.js with Node.js

RDKit.js works server-side for cheminformatics computations — no browser needed. Use ESM imports directly from the npm package.

### Setup

```bash
npm i @rdkit/rdkit
```

Your `package.json` must set `"type": "module"` (or use `.mjs` extension):

```json
{
  "type": "module"
}
```

### SVG drawing

```js
import initRDKitModule from "@rdkit/rdkit";

let rdkit = await initRDKitModule();
let smiles = "CC(=O)Oc1ccccc1C(=O)O";
let mol = rdkit.get_mol(smiles);
let svg = mol.get_svg();
console.log(svg);
mol.delete();
```

### Descriptors calculation

```js
import initRDKitModule from "@rdkit/rdkit";

let rdkit = await initRDKitModule();
let mol = rdkit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
let descriptors = JSON.parse(mol.get_descriptors());
console.log(descriptors.amw);          // molecular weight
console.log(descriptors.CrippenClogP); // LogP
mol.delete();
```

### Substructure search with highlighting

```js
import initRDKitModule from "@rdkit/rdkit";

let rdkit = await initRDKitModule();
let smiles = "CC(=O)Oc1ccccc1C(=O)O";
let mol = rdkit.get_mol(smiles);
let qmol = rdkit.get_qmol("O=C");
let mdetails = mol.get_substruct_match(qmol);
let svg = mol.get_svg_with_highlights(mdetails);
console.log(svg);
mol.delete();
qmol.delete();
```

### Custom atom highlighting + drawing options

```js
import initRDKitModule from "@rdkit/rdkit";

let rdkit = await initRDKitModule();
let mol = rdkit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
let qmol = rdkit.get_qmol("O=C");

let mdetails = JSON.parse(mol.get_substruct_match(qmol));
mdetails.highlightColour = [1, 0, 1];
mdetails.legend = 'aspirin';
mdetails.addAtomIndices = true;

let svg = mol.get_svg_with_highlights(JSON.stringify(mdetails));
console.log(svg);
mol.delete();
qmol.delete();
```

### Coordinate alignment with template

```js
import initRDKitModule from "@rdkit/rdkit";

let rdkit = await initRDKitModule();
let mol = rdkit.get_mol("c1cnc(C)nc1C(=O)O");

let template = `
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
`;

let qmol = rdkit.get_mol(template);
mol.generate_aligned_coords(qmol, true);
let tdetails = mol.get_substruct_match(qmol);
let svg = mol.get_svg_with_highlights(tdetails);
console.log(svg);
mol.delete();
qmol.delete();
```

### Identifiers generation

Generate multiple molecular representations from a single SMILES:

```js
import initRDKitModule from "@rdkit/rdkit";

let rdkit = await initRDKitModule();
let smiles = "CC(=O)Oc1ccccc1C(=O)O";
let mol = rdkit.get_mol(smiles);

console.log("SMILES:", mol.get_smiles());
console.log("CXSMILES:", mol.get_cxsmiles());
console.log("InChI:", mol.get_inchi());
console.log("MolBlock:", mol.get_molblock());
console.log("V3KMolBlock:", mol.get_v3Kmolblock());
console.log("Morgan FP:", mol.get_morgan_fp_as_binary_text(2, 128));
console.log("Pattern FP:", mol.get_pattern_fp_as_binary_text(2, 128));
console.log("Aromatic:", mol.get_aromatic_form());
console.log("Kekule:", mol.get_kekule_form());

mol.delete();
```

### Key differences from browser

1. **Direct import**: `import initRDKitModule from "@rdkit/rdkit"` — no script tag needed
2. **Top-level await**: Node.js ESM supports top-level `await`
3. **No Canvas API**: Canvas drawing (`draw_to_canvas`) requires a browser. Use SVG (`get_svg`, `get_svg_with_highlights`) instead
4. **No global window**: The RDKit instance is a local variable, not attached to `window`

### Use cases for Node.js

- Server-side molecule rendering (generate SVGs in batch)
- Cheminformatics pipelines (descriptors, fingerprints, InChI generation)
- Build-time molecule asset generation
- Automated testing of cheminformatics workflows



