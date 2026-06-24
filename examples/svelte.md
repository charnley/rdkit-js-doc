---
layout: page
title: Svelte
permalink: /examples/svelte/
---

## Using RDKit.js with Svelte

A reusable `MoleculeStructure` component using Svelte's reactive declarations and lifecycle hooks.

### Setup

Load the RDKit script in `public/index.html` (SvelteKit) or `index.html` (vanilla Svelte):

```html
<head>
  <script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
</head>
```

### initRDKit singleton

```js
// src/lib/initRDKit.js
export const initRDKit = (() => {
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
```

### MoleculeStructure.svelte

{% raw %}
```svelte
<script>
  import { onMount, afterUpdate } from "svelte";
  import { initRDKit } from "../lib/initRDKit.js";

  export let id;
  export let structure = "";
  export let subStructure = "";
  export let svgMode = false;
  export let width = 250;
  export let height = 200;
  export let extraDetails = {};
  export let drawingDelay = undefined;

  let rdKitLoaded = false;
  let rdKitError = false;
  let svgContent = "";

  const defaultDetails = {
    width,
    height,
    bondLineWidth: 1,
    addStereoAnnotation: true,
    ...extraDetails
  };

  $: molDetails = {
    width,
    height,
    bondLineWidth: 1,
    addStereoAnnotation: true,
    ...extraDetails
  };

  function isValid(mol) {
    return !!mol;
  }

  function getMolDetails(mol, qmol) {
    if (isValid(mol) && isValid(qmol)) {
      const matches = JSON.parse(mol.get_substruct_matches(qmol) || "[]");
      const merged = matches.reduce(
        (acc, { atoms, bonds }) => ({
          atoms: [...acc.atoms, ...atoms],
          bonds: [...acc.bonds, ...bonds]
        }),
        { atoms: [], bonds: [] }
      );
      return JSON.stringify({ ...molDetails, ...merged });
    }
    return JSON.stringify(molDetails);
  }

  function draw() {
    if (!rdKitLoaded || rdKitError) return;

    const mol = window.RDKit.get_mol(structure || "invalid");
    const qmol = window.RDKit.get_qmol(subStructure || "invalid");

    if (svgMode && isValid(mol)) {
      const details = getMolDetails(mol, qmol);
      svgContent = mol.get_svg_with_highlights(details);
    } else if (isValid(mol)) {
      const canvas = document.getElementById(id);
      if (canvas) {
        const details = getMolDetails(mol, qmol);
        mol.draw_to_canvas_with_highlights(canvas, details);
      }
    }

    mol?.delete();
    qmol?.delete();
  }

  onMount(() => {
    initRDKit()
      .then(() => {
        rdKitLoaded = true;
        draw();
      })
      .catch(() => {
        rdKitError = true;
      });
  });

  // Redraw when props change
  $: if (rdKitLoaded && (structure || subStructure || svgMode || width || height)) {
    if (drawingDelay) {
      setTimeout(draw, drawingDelay);
    } else {
      draw();
    }
  }

  // Canvas needs redraw on update since DOM replaces canvas element
  afterUpdate(() => {
    if (rdKitLoaded && !svgMode) {
      draw();
    }
  });
</script>

{#if rdKitError}
  <p>Error loading renderer</p>
{:else if !rdKitLoaded}
  <p>Loading renderer...</p>
{:else if !isValid(window.RDKit?.get_mol(structure || "invalid"))}
  <span title="Cannot render structure: {structure}">Render Error.</span>
{:else if svgMode}
  <div class="molecule-structure-svg" style="width:{width}px;height:{height}px">
    {@html svgContent}
  </div>
{:else}
  <div class="molecule-canvas-container">
    <canvas {id} {width} {height} title={structure}></canvas>
  </div>
{/if}

<style>
  :global(.molecule-structure-svg svg rect:first-of-type) {
    fill: transparent !important;
  }
</style>
```
{% endraw %}

### Usage

{% raw %}
```svelte
<script>
  import MoleculeStructure from "./MoleculeStructure.svelte";
</script>

<!-- Canvas rendering -->
<MoleculeStructure
  id="mol1"
  structure="CC(=O)Oc1ccccc1C(=O)O"
  width={300}
  height={250}
/>

<!-- SVG mode -->
<MoleculeStructure
  id="mol2"
  structure="CC(=O)Oc1ccccc1C(=O)O"
  svgMode={true}
  width={300}
  height={250}
/>

<!-- Substructure search -->
<MoleculeStructure
  id="mol3"
  structure="c1cc(O)ccc1C(=O)O"
  subStructure="c1ccccc1"
  width={300}
  height={250}
/>

<!-- With legend -->
<MoleculeStructure
  id="mol4"
  structure="CC(=O)Oc1ccccc1C(=O)O"
  extraDetails={{ legend: "Aspirin", addAtomIndices: true }}
  width={300}
  height={250}
/>
```
{% endraw %}

### SvelteKit notes

If using SvelteKit, wrapper components that use browser-only APIs must be client-side only. Two approaches:

**Option 1: Dynamic import with `browser` check**

{% raw %}
```svelte
<script>
  import { browser } from "$app/environment";
  import { onMount } from "svelte";

  let MoleculeStructure;

  onMount(async () => {
    const mod = await import("./MoleculeStructure.svelte");
    MoleculeStructure = mod.default;
  });
</script>

{#if MoleculeStructure}
  <svelte:component this={MoleculeStructure}
    id="mol1"
    structure="CC(=O)Oc1ccccc1C(=O)O"
  />
{/if}
```
{% endraw %}

**Option 2: Use `+layout.js` with `ssr = false`**

```js
// src/routes/+layout.js
export const ssr = false;
```

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `id` | string | required | Unique ID for canvas element |
| `structure` | string | required | SMILES string to render |
| `subStructure` | string | `""` | SMARTS for substructure highlighting |
| `svgMode` | boolean | `false` | Render as SVG instead of canvas |
| `width` | number | `250` | Width in pixels |
| `height` | number | `200` | Height in pixels |
| `extraDetails` | object | `{}` | Extra `MolDrawOptions` to merge |
| `drawingDelay` | number | `undefined` | Delay before drawing (ms) |

### Key Svelte patterns

1. **Reactive declarations** (`$:`): Automatically redraw when props change
2. **`afterUpdate`**: Canvas requires redraw after DOM updates since Svelte replaces elements
3. **`{@html}`**: Renders raw SVG string — same as `dangerouslySetInnerHTML` in React
4. **`onMount`**: Initialize RDKit once when component first mounts (runs only client-side)

