---
layout: page
title: Using RDKit.js with Vue
menu: Vue
permalink: /examples/vue/
---

Minimal Vue 3 + TypeScript + Vite app demonstrating RDKit.js usage.

```bash
npm create vite@latest project_name -- --template vue-ts
cd project_name
npm install @rdkit/rdkit
```

No Vite config change is needed — `?url` is default Vite behavior.
The whole WASM wiring lives in `src/rdkit.ts` below: `?url` resolves the `.wasm` to a URL string (dev: `node_modules/...`; prod: hashed asset), and `locateFile` tells emscripten's loader to fetch it there.
The `locateFile` override is the only WASM-specific bit, without it emscripten looks next to `import.meta.url` which does not exist after bundling.

```ts
// src/rdkit.ts
import initRDKitModule, { type MainModule, type Mol } from '@rdkit/rdkit'
import wasmUrl from '@rdkit/rdkit/RDKit_minimal.wasm?url'

export type { MainModule, Mol }

let rdkit: MainModule | null = null
let loading: Promise<MainModule> | null = null

export function getRDKit(): Promise<MainModule> {
  if (rdkit) return Promise.resolve(rdkit)
  if (!loading) loading = initRDKitModule({ locateFile: () => wasmUrl })
  return loading.then((m) => {
    rdkit = m
    return m
  })
}
```

`getRDKit()` instantiates the WASM module once and reuses it across every caller.
Example structure render using the rdkit object from ´getRDKit' could be something like

```vue
// src/components/MoleculeStructure.vue
<script setup lang="ts">
import { ref, watch, onBeforeUnmount } from 'vue'
import { getRDKit, type Mol } from '../rdkit'

const props = withDefaults(defineProps<{
  smiles: string
  width?: number
  height?: number
}>(), { width: 300, height: 300 })

const svg = ref('')
const loaded = ref(false)
const error = ref('')

let mol: Mol | null = null

async function draw() {
  error.value = ''
  try {
    const rdkit = await getRDKit()
    mol?.delete()
    mol = rdkit.get_mol(props.smiles || 'invalid')
    if (!mol) {
      loaded.value = false
      error.value = `Cannot parse: ${props.smiles}`
      return
    }
    svg.value = mol.get_svg_with_highlights(
      JSON.stringify({ width: props.width, height: props.height }),
    )
    loaded.value = true
  } catch (e) {
    loaded.value = false
    error.value = e instanceof Error ? e.message : String(e)
  }
}

watch(() => props.smiles, draw, { immediate: true })

onBeforeUnmount(() => mol?.delete())
</script>

<template>
  <p v-if="error" class="error">{{ error }}</p>
  <p v-else-if="!loaded">Loading…</p>
  <div
    v-else
    class="mol-svg"
    :style="{ width: `${width}px`, height: `${height}px` }"
    v-html="svg"
  ></div>
</template>

<style scoped>
.mol-svg :deep(svg) { width: 100%; height: 100%; }
.mol-svg :deep(svg rect:first-of-type) { fill: transparent !important; }
.error { color: #c0392b; }
</style>
```

Example usage

```vue
// `src/App.vue`
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import MoleculeStructure from './components/MoleculeStructure.vue'
import { getRDKit } from './rdkit.ts'

const smiles = ref('CC(=O)Oc1ccccc1C(=O)O')
const version = ref('')

onMounted(async () => {
  version.value = (await getRDKit()).version()
})
</script>

<template>
  <main class="app">
    <h1>RDKit.js + Vue</h1>
    <p v-if="version">RDKit version: <code>{{ version }}</code></p>
    <p v-else>Loading RDKit WASM…</p>

    <label>
      SMILES
      <input v-model="smiles" spellcheck="false" placeholder="e.g. CCO" />
    </label>

    <MoleculeStructure :smiles="smiles" :width="300" :height="250" />
  </main>
</template>

<style scoped>
.app { display: flex; flex-direction: column; align-items: center; gap: 20px; padding: 40px 20px; }
label { display: flex; flex-direction: column; gap: 6px; align-items: center; }
input { font: 15px ui-monospace, Consolas, monospace; padding: 8px 12px; width: 320px; }
</style>
```

Which will show a molecule from the vue scaffold setup.

