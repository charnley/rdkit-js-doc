---
layout: page
title: Using RDKit.js with React
menu: React
permalink: /examples/react/
---

RDKit.js loads a `.wasm` module asynchronously. The simplest setup uses [Vite](https://vite.dev) which handles `.wasm` files natively.

Install dependencies:

```sh
npm install react react-dom @rdkit/rdkit
npm install -D vite @vitejs/plugin-react
```

Tell Vite to treat `.wasm` files as assets so it serves them from `node_modules` in dev and copies them to `dist/` in production:

```js
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  assetsInclude: ["**/*.wasm"],
});
```

Import `initRDKitModule` directly from `@rdkit/rdkit`.
It returns a Promise that resolves once the `.wasm` file is fetched and compiled.
Call `get_mol()` with a SMILES string and use `dangerouslySetInnerHTML` to render the SVG:

```jsx
// src/App.jsx
import { useEffect, useState } from "react";
import { createRoot } from "react-dom/client";
import initRDKitModule from "@rdkit/rdkit";
import wasmUrl from "@rdkit/rdkit/dist/RDKit_minimal.wasm?url";
import "./App.css";

function App() {
  const [RDKit, setRDKit] = useState(null);

  useEffect(() => {
    initRDKitModule({ locateFile: () => wasmUrl }).then(setRDKit);
  }, []);

  if (!RDKit) return <div className="loading">Loading RDKit…</div>;

  const mol = RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
  const svg = mol.get_svg_with_highlights(
    JSON.stringify({ width: 300, height: 300})
  );
  mol.delete();

  return (
    <div className="app">
      <h1>RDKit + React</h1>
      <div className="mol" dangerouslySetInnerHTML={{ __html: svg }} />
    </div>
  );
}

createRoot(document.getElementById("root")).render(<App />);
```

