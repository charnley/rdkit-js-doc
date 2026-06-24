---
layout: page
title: Using RDKit.js with React
menu: React
permalink: /examples/react/
---

A reusable `MoleculeStructure` component handles RDKit loading, canvas/SVG rendering, and substructure highlighting.

### Setup

Load the RDKit script in `public/index.html`:

```html
<head>
  <script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
</head>
```

Or copy `RDKit_minimal.js` and `RDKit_minimal.wasm` from `node_modules/@rdkit/rdkit/dist/` to your `public/` folder and reference locally.

### initRDKit singleton

`src/utils/initRDKit.js`:

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

### MoleculeStructure component

{% raw %}
```jsx
import React, { Component } from "react";
import PropTypes from "prop-types";
import initRDKit from "../../utils/initRDKit";

class MoleculeStructure extends Component {
  static propTypes = {
    id: PropTypes.string.isRequired,
    className: PropTypes.string,
    svgMode: PropTypes.bool,
    width: PropTypes.number,
    height: PropTypes.number,
    structure: PropTypes.string.isRequired,
    subStructure: PropTypes.string,
    extraDetails: PropTypes.object,
    drawingDelay: PropTypes.number
  };

  static defaultProps = {
    subStructure: "",
    className: "",
    width: 250,
    height: 200,
    svgMode: false,
    extraDetails: {},
    drawingDelay: undefined
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

  draw() {
    if (this.props.drawingDelay) {
      setTimeout(() => this.drawSVGorCanvas(), this.props.drawingDelay);
    } else {
      this.drawSVGorCanvas();
    }
  }

  drawSVGorCanvas() {
    const mol = window.RDKit.get_mol(this.props.structure || "invalid");
    const qmol = window.RDKit.get_qmol(this.props.subStructure || "invalid");

    if (this.props.svgMode && mol) {
      const svg = mol.get_svg_with_highlights(this.getMolDetails(mol, qmol));
      this.setState({ svg });
    } else if (mol) {
      const canvas = document.getElementById(this.props.id);
      mol.draw_to_canvas_with_highlights(
        canvas,
        this.getMolDetails(mol, qmol)
      );
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
      return JSON.stringify({
        ...this.MOL_DETAILS,
        ...(this.props.extraDetails || {}),
        ...merged
      });
    }
    return JSON.stringify({
      ...this.MOL_DETAILS,
      ...(this.props.extraDetails || {})
    });
  }

  componentDidMount() {
    initRDKit()
      .then(() => {
        this.setState({ rdKitLoaded: true });
        this.draw();
      })
      .catch(() => {
        this.setState({ rdKitError: true });
      });
  }

  componentDidUpdate(prevProps) {
    if (this.state.rdKitLoaded) {
      const changed =
        prevProps.structure !== this.props.structure ||
        prevProps.subStructure !== this.props.subStructure ||
        prevProps.svgMode !== this.props.svgMode ||
        prevProps.width !== this.props.width ||
        prevProps.height !== this.props.height;
      if (changed) this.draw();
    }
  }

  render() {
    if (this.state.rdKitError) return "Error loading renderer.";
    if (!this.state.rdKitLoaded) return "Loading renderer...";

    const mol = window.RDKit.get_mol(this.props.structure || "invalid");
    const isValid = !!mol;
    mol?.delete();

    if (!isValid) {
      return (
        <span title={`Cannot render: ${this.props.structure}`}>
          Render Error.
        </span>
      );
    }

    if (this.props.svgMode) {
      return (
        <div
          className={"molecule-structure-svg " + (this.props.className || "")}
          style={{ width: this.props.width, height: this.props.height }}
          dangerouslySetInnerHTML={{ __html: this.state.svg }}
        />
      );
    }

    return (
      <div className={"molecule-canvas-container " + (this.props.className || "")}>
        <canvas
          title={this.props.structure}
          id={this.props.id}
          width={this.props.width}
          height={this.props.height}
        />
      </div>
    );
  }
}

export default MoleculeStructure;
```
{% endraw %}

### Usage

{% raw %}
```jsx
import MoleculeStructure from "./components/MoleculeStructure";

function App() {
  return (
    <div>
      {/* Canvas rendering */}
      <MoleculeStructure
        id="mol1"
        structure="CC(=O)Oc1ccccc1C(=O)O"
        width={300}
        height={250}
      />

      {/* SVG rendering */}
      <MoleculeStructure
        id="mol2"
        structure="c1ccccc1"
        subStructure="c1ccccc1"
        svgMode={true}
        width={300}
        height={250}
      />

      {/* With legend */}
      <MoleculeStructure
        id="mol3"
        structure="CC(=O)Oc1ccccc1C(=O)O"
        extraDetails={{ legend: "Aspirin", addAtomIndices: true }}
        width={300}
        height={250}
      />

      {/* Substructure highlight */}
      <MoleculeStructure
        id="mol4"
        structure="c1cc(O)ccc1C(=O)O"
        subStructure="c1ccccc1"
        width={300}
        height={250}
      />
    </div>
  );
}
```
{% endraw %}

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `id` | string | required | Unique ID for the canvas element |
| `structure` | string | required | SMILES string to render |
| `subStructure` | string | `""` | SMARTS for substructure highlighting |
| `svgMode` | bool | `false` | Render as SVG instead of canvas |
| `width` | number | `250` | Width in pixels |
| `height` | number | `200` | Height in pixels |
| `extraDetails` | object | `{}` | Extra `MolDrawOptions` to merge |
| `className` | string | `""` | CSS class for container |
| `drawingDelay` | number | `undefined` | Delay before drawing (ms) |

### SVG background transparency

Add this CSS to remove the default white background on SVG renderings:

```css
.molecule-structure-svg svg rect:first-of-type {
  fill: transparent !important;
}
```



