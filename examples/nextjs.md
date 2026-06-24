---
layout: page
title: Next.js
permalink: /examples/nextjs/
---

## Using RDKit.js with Next.js

Next.js requires special handling since RDKit.js uses WebAssembly and client-side only APIs.

### Setup

Install the package:

```bash
npm i @rdkit/rdkit
```

### Copy WASM file

Use a custom webpack config to copy the WASM file to your public directory:

```js
// next.config.js
const CopyWebpackPlugin = require("copy-webpack-plugin");

module.exports = {
  webpack: (config) => {
    config.plugins.push(
      new CopyWebpackPlugin({
        patterns: [
          {
            from: "node_modules/@rdkit/rdkit/dist/RDKit_minimal.wasm",
            to: "static/chunks/pages"
          }
        ]
      })
    );
    return config;
  }
};
```

### MoleculeStructure component

Because RDKit relies on `window` and DOM APIs, use dynamic import with `ssr: false`:

{% raw %}
```jsx
import React, { Component } from "react";
import PropTypes from "prop-types";
import initRDKitModule from "@rdkit/rdkit";

const initRDKit = (() => {
  let rdkitLoadingPromise;

  return () => {
    if (!rdkitLoadingPromise) {
      rdkitLoadingPromise = new Promise((resolve, reject) => {
        initRDKitModule()
          .then((RDKit) => resolve(RDKit))
          .catch(() => reject());
      });
    }
    return rdkitLoadingPromise;
  };
})();

class MoleculeStructure extends Component {
  static propTypes = {
    id: PropTypes.string.isRequired,
    structure: PropTypes.string.isRequired,
    subStructure: PropTypes.string,
    svgMode: PropTypes.bool,
    width: PropTypes.number,
    height: PropTypes.number,
    extraDetails: PropTypes.object
  };

  static defaultProps = {
    subStructure: "",
    svgMode: false,
    width: 250,
    height: 200,
    extraDetails: {}
  };

  constructor(props) {
    super(props);
    this.MOL_DETAILS = {
      width: this.props.width,
      height: this.props.height,
      bondLineWidth: 1,
      addStereoAnnotation: true,
      ...this.props.extraDetails
    };
    this.state = {
      svg: undefined,
      rdKitLoaded: false,
      rdKitError: false
    };
  }

  componentDidMount() {
    initRDKit()
      .then((RDKit) => {
        this.RDKit = RDKit;
        this.setState({ rdKitLoaded: true });
        this.draw();
      })
      .catch(() => this.setState({ rdKitError: true }));
  }

  draw() {
    const mol = this.RDKit.get_mol(this.props.structure || "invalid");
    const qmol = this.RDKit.get_qmol(this.props.subStructure || "invalid");

    if (this.props.svgMode && mol) {
      const svg = mol.get_svg_with_highlights(this.getMolDetails(mol, qmol));
      this.setState({ svg });
    } else if (mol) {
      const canvas = document.getElementById(this.props.id);
      mol.draw_to_canvas_with_highlights(canvas, this.getMolDetails(mol, qmol));
    }

    mol?.delete();
    qmol?.delete();
  }

  getMolDetails(mol, qmol) {
    if (mol && qmol) {
      const matches = JSON.parse(mol.get_substruct_matches(qmol));
      const merged = matches.reduce(
        (acc, { atoms, bonds }) => ({
          atoms: [...acc.atoms, ...atoms],
          bonds: [...acc.bonds, ...bonds]
        }),
        { bonds: [], atoms: [] }
      );
      return JSON.stringify({ ...this.MOL_DETAILS, ...merged });
    }
    return JSON.stringify(this.MOL_DETAILS);
  }

  render() {
    if (this.state.rdKitError) return "Error loading renderer.";
    if (!this.state.rdKitLoaded) return "Loading renderer...";

    if (this.props.svgMode) {
      return (
        <div
          className="molecule-structure-svg"
          style={{ width: this.props.width, height: this.props.height }}
          dangerouslySetInnerHTML={{ __html: this.state.svg }}
        />
      );
    }

    return (
      <canvas
        id={this.props.id}
        width={this.props.width}
        height={this.props.height}
      />
    );
  }
}

export default MoleculeStructure;
```
{% endraw %}

### Dynamic import with SSR disabled

In your page component, import MoleculeStructure dynamically:

```jsx
// pages/index.js
import dynamic from "next/dynamic";

const MoleculeStructure = dynamic(
  () => import("../components/MoleculeStructure/MoleculeStructure"),
  { ssr: false }
);

export default function Home() {
  return (
    <div>
      <MoleculeStructure
        id="mol1"
        structure="CC(=O)Oc1ccccc1C(=O)O"
        width={300}
        height={250}
      />
    </div>
  );
}
```

### Key differences from React

1. **Module import**: Uses `import initRDKitModule from "@rdkit/rdkit"` directly (Node import), not `window.initRDKitModule` from a script tag
2. **RDKit on instance**: Store the RDKit instance on `this.RDKit` instead of `window.RDKit` — avoids polluting global scope in SSR context
3. **No SSR**: The component must only render on the client. Use `dynamic(() => import(...), { ssr: false })`
4. **WASM path**: The `locateFile` option may be needed if the WASM file isn't auto-detected:

```js
initRDKitModule({ locateFile: () => "/static/chunks/pages/RDKit_minimal.wasm" })
```



