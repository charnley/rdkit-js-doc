---
layout: page
title: Using RDKit.js with Vue
menu: Vue
permalink: /examples/vue/
---

A `MoleculeStructure` component using Vue 3 Composition API (`<script setup>`) with TypeScript support.

### Setup

Load the RDKit script in `index.html`:

```html
<head>
  <script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
</head>
```

### TypeScript declarations

Declare the global types:

```typescript
// globals.d.ts
import { RDKitModule, RDKitLoader } from "@rdkit/rdkit";

declare global {
  interface Window {
    RDKit: RDKitModule;
    initRDKitModule: RDKitLoader;
  }
}
```

### initRDKit singleton

`src/utils/initRDKit.ts`:

```typescript
import { RDKitModule } from "@rdkit/rdkit";

const initRDKit = (() => {
  let rdkitLoadingPromise: Promise<RDKitModule>;

  return () => {
    if (!rdkitLoadingPromise) {
      rdkitLoadingPromise = new Promise((resolve, reject) => {
        window
          .initRDKitModule()
          .then((RDKit) => {
            window.RDKit = RDKit;
            resolve(RDKit);
          })
          .catch((e) => {
            reject();
          });
      });
    }
    return rdkitLoadingPromise;
  };
})();

export default initRDKit;
```

### MoleculeStructure.vue

```vue
<template>
  <p v-if="rdkitError">Error loading renderer</p>
  <p v-if="!rdkitLoaded">Loading renderer</p>

  <span
    v-else-if="!isValidMol"
    :title="`Cannot render structure: ${structure}`"
  />

  <div
    v-else-if="svgMode"
    :class="`molecule-structure-svg ${className}`"
    :style="{ width: `${width}px`, height: `${height}px` }"
    v-html="svg"
  />

  <div v-else :class="`molecule-canvas-container ${className}`">
    <canvas
      :title="structure"
      :id="id"
      :width="width"
      :height="height"
    />
  </div>
</template>

<script setup lang="ts">
import { nextTick, onMounted, reactive, ref, watch } from "vue";
import { JSMol } from "@rdkit/rdkit";
import initRDKit from "../utils/initRDKit";

const props = defineProps({
  id: { type: String, required: true },
  className: { type: String, default: "" },
  svgMode: { type: Boolean, default: false },
  width: { type: Number, default: 250 },
  height: { type: Number, default: 200 },
  structure: { type: String, required: true },
  subStructure: { type: String, default: "" },
  extraDetails: { type: Object, default: {} },
  drawingDelay: { type: Number, default: undefined }
});

let rdkitLoaded = ref(false);
let rdkitError = ref(false);
const svg = ref("");
const molDetails = reactive({
  width: props.width,
  height: props.height,
  bondLineWidth: 1,
  addStereoAnnotation: true,
  ...props.extraDetails
});

const isValidMol = ref(true);

function isValid(m: JSMol | null) {
  return !!m;
}

function getMolDetails(mol: JSMol | null, qmol: JSMol | null) {
  if (isValid(mol) && isValid(qmol)) {
    const details = JSON.parse(mol!.get_substruct_matches(qmol!) || "[]");
    const merged = details.length
      ? details.reduce(
          (acc: any, { atoms, bonds }: any) => ({
            atoms: [...acc.atoms, ...atoms],
            bonds: [...acc.bonds, ...bonds]
          }),
          { atoms: [], bonds: [] }
        )
      : details;
    return JSON.stringify({
      ...molDetails,
      ...(props.extraDetails || {}),
      ...merged
    });
  }
  return JSON.stringify({
    ...molDetails,
    ...(props.extraDetails || {})
  });
}

async function draw() {
  const mol = window.RDKit.get_mol(props.structure || "invalid");
  const qmol = window.RDKit.get_qmol(props.subStructure || "invalid");

  isValidMol.value = isValid(mol);

  if (props.svgMode && isValidMol.value) {
    svg.value = mol!.get_svg_with_highlights(getMolDetails(mol, qmol));
  } else if (isValidMol.value) {
    await nextTick();
    const canvas = document.getElementById(props.id) as HTMLCanvasElement;
    mol!.draw_to_canvas_with_highlights(canvas, getMolDetails(mol, qmol));
  }

  mol?.delete();
  qmol?.delete();
}

onMounted(() => {
  initRDKit()
    .then(() => {
      rdkitLoaded.value = true;
      draw();
    })
    .catch(() => {
      rdkitError.value = true;
    });
});

watch(props, () => {
  if (rdkitLoaded.value) draw();
});
</script>

<style>
.molecule-structure-svg svg rect:first-of-type {
  fill: transparent !important;
}
</style>
```

### Usage

```vue
<template>
  <div>
    <MoleculeStructure
      id="mol1"
      structure="CC(=O)Oc1ccccc1C(=O)O"
      :width="300"
      :height="250"
    />
    <MoleculeStructure
      id="mol2"
      structure="c1ccccc1"
      subStructure="c1ccccc1"
      :svgMode="true"
      :width="300"
      :height="250"
    />
  </div>
</template>

<script setup lang="ts">
import MoleculeStructure from "./components/MoleculeStructure.vue";
</script>
```

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `id` | String | required | Unique ID for canvas element |
| `structure` | String | required | SMILES string to render |
| `subStructure` | String | `""` | SMARTS for substructure highlighting |
| `svgMode` | Boolean | `false` | Render as SVG instead of canvas |
| `width` | Number | `250` | Width in pixels |
| `height` | Number | `200` | Height in pixels |
| `extraDetails` | Object | `{}` | Extra `MolDrawOptions` to merge |
| `className` | String | `""` | CSS class for container |
| `drawingDelay` | Number | `undefined` | Delay before drawing (ms) |



