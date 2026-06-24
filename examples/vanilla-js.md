---
layout: page
title: Vanilla JavaScript
permalink: /examples/vanilla-js/
---

## Using RDKit.js with plain JavaScript

The simplest way to use RDKit.js. Load the script tag, initialize the module, and start drawing molecules.

### Setup

Add this to your `index.html`:

```html
<head>
  <script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
  <script>
    window
      .initRDKitModule()
      .then(function (RDKit) {
        window.RDKit = RDKit;
        console.log("RDKit version: " + RDKit.version());
      });
  </script>
</head>
```

### Singleton loader

Create a utility to ensure RDKit loads only once:

```js
const initRDKit = (() => {
  let rdkitLoadingPromise;

  return () => {
    if (!rdkitLoadingPromise) {
      rdkitLoadingPromise = new Promise((resolve, reject) => {
        window
          .initRDKitModule()
          .then((RDKit) => {
            window.RDKit = RDKit;
            resolve(RDKit);
          })
          .catch(reject);
      });
    }
    return rdkitLoadingPromise;
  };
})();

// Usage:
initRDKit().then(RDKit => { /* ready */ });
```

### SVG rendering

```html
<div id="molecule-svg"></div>
```

```js
initRDKit().then(() => {
  var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
  var svg = mol.get_svg();
  document.getElementById("molecule-svg").innerHTML = svg;
  mol.delete();
});
```

### Canvas rendering

```html
<canvas id="molecule-canvas" width="250" height="200"></canvas>
```

```js
initRDKit().then(() => {
  var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
  var canvas = document.getElementById("molecule-canvas");
  mol.draw_to_canvas(canvas, -1, -1);
  mol.delete();
});
```

### Substructure search with highlighting

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var smarts = "Oc1[c,n]cccc1";
var qmol = window.RDKit.get_qmol(smarts);
var mdetails = mol.get_substruct_match(qmol);
mol.draw_to_canvas_with_highlights(canvas, mdetails);
mol.delete();
qmol.delete();
```

The same works for SVG:

```js
var svg = mol.get_svg_with_highlights(mdetails);
```

### Custom drawing options

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var details = {};
details.atoms = [0, 1, 10];
details.explicitMethyl = true;
details.addAtomIndices = true;
details.legend = 'aspirin';
mol.draw_to_canvas_with_highlights(canvas, JSON.stringify(details));
mol.delete();
```

### Combine substructure match + custom options

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var qmol = window.RDKit.get_qmol("O=C");
var mdetails = JSON.parse(mol.get_substruct_match(qmol));
mdetails.highlightColour = [1, 0, 1];
mdetails.legend = 'aspirin';
mol.draw_to_canvas_with_highlights(canvas, JSON.stringify(mdetails));
mol.delete();
qmol.delete();
```

### Template coordinate alignment

Constrain molecule coordinates to a template's shape. Useful for displaying molecule grids with consistent core orientation.

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

### Descriptors

```js
var mol = window.RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
var descriptors = JSON.parse(mol.get_descriptors());
console.log(descriptors.amw);        // molecular weight
console.log(descriptors.CrippenClogP); // LogP
var fp = mol.get_morgan_fp(2, 128);  // Morgan fingerprint
mol.delete();
```

### Drawing options reference

The `MolDrawOptions` supported via `get_svg_with_highlights()` and `draw_to_canvas_with_highlights()`:

| Option | Type | Description |
|--------|------|-------------|
| `width`, `height` | number | SVG dimensions |
| `offsetx`, `offsety` | number | Canvas subregion offset |
| `atoms` | number[] | Atoms to highlight |
| `bonds` | number[] | Bonds to highlight |
| `legend` | string | Legend text under molecule |
| `legendFontSize` | number | Legend font size |
| `bondLineWidth` | number | Bond line thickness |
| `addAtomIndices` | boolean | Show atom numbers |
| `addBondIndices` | boolean | Show bond numbers |
| `addStereoAnnotation` | boolean | Show stereo labels |
| `explicitMethyl` | boolean | Show terminal methyl groups |
| `highlightColour` | [r,g,b] | Highlight color (0-1 range) |
| `fillHighlights` | boolean | Fill highlighted atoms |
| `continuousHighlight` | boolean | Continuous highlight rings |
| `atomHighlightsAreCircles` | boolean | Circle highlights on atoms |
| `clearBackground` | boolean | Transparent background |
| `rotate` | number | Rotation in degrees |
| `fixedBondLength` | number | Fixed bond pixel length |
| `padding` | number | Padding around molecule |
| `backgroundColour` | [r,g,b] | Background color |
| `atomLabels` | object | Custom atom label map |

Always call `mol.delete()` when done — RDKit uses C++ memory that must be freed manually.



