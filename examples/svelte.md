---
layout: page
title: Using RDKit.js with Svelte
menu: Svelte
permalink: /examples/svelte/
---


Vite (the sveltekit bundler) needs `assetsInclude` to handle `.wasm` files.

```js
// vite.config.js
export default {
  assetsInclude: ['**/*.wasm']
};
```

Import the JavaScript module and WASM binary with the `?url` suffix, then Vite compiles it to a hashed URL at build time so the browser always finds it.

```js
// src/lib/rdkitUtils.js
import initRDKitModule from '@rdkit/rdkit';
import wasmUrl from '@rdkit/rdkit/RDKit_minimal.wasm?url';

let rdkit = null;
let loading = null;

export async function getRDKit() {
  if (rdkit) return rdkit;
  if (!loading) {
    promise = initRDKitModule({ locateFile: () => wasmUrl });
  }
  rdkit = await promise;
  return rdkit;
}
```

Which means you can use 

```js
// src/routes/test/+page.svelte
import { getRDKit } from '$lib/rdkitUtils.js';
```

### Example Molecule Render

Minimal Svelte 5 component that renders a molecule as SVG.

{% raw %}
```svelte
<!-- src/routes/example-page/MoleculeStructure.svelte -->
<script>
  import { getRDKit } from '$lib/rdkitUtils.js';

  let { structure = '', width = 250, height = 200 } = $props();

  let svg = $state('');
  let loaded = $state(false);
  let error = $state('');

  $effect(() => {
    structure; width; height;
    loadAndDraw();
  });

  async function loadAndDraw() {
    try {
      const rdkit = await getRDKit();
      const mol = rdkit.get_mol(structure || 'invalid');

      if (!mol) {
        error = `Cannot parse: ${structure}`;
        return
      }

      svg = mol.get_svg_with_highlights(JSON.stringify({ width, height }));
      mol.delete();
      loaded = true;
      error = '';

    } catch (e) {
      error = e.message;
    }
  }

  loadAndDraw();
</script>

{#if error}
  <p class="error">{error}</p>
{:else if !loaded}
  <p>Loading…</p>
{:else}
  <div class="mol-svg" style="width:{width}px;height:{height}px">{@html svg}</div>
{/if}

<style>
  .mol-svg :global(svg) { width: 100%; height: 100%; }
</style>
```
{% endraw %}

{% raw %}
```svelte
<!-- src/routes/example-page/+page.svelte -->
<script>
  import MoleculeStructure from './MoleculeStructure.svelte';
</script>

<MoleculeStructure structure="CC(=O)Oc1ccccc1C(=O)O" width={300} height={250} />
```
{% endraw %}

