---
layout: page
title: Using RDKit.js with Angular
menu: Angular
permalink: /examples/angular/
---


Two approaches: CDN script tag or NPM distribution with Angular's build config.

### Option 1: CDN (recommended)

Add to `src/index.html`:

```html
<head>
  <script src="https://unpkg.com/@rdkit/rdkit/dist/RDKit_minimal.js"></script>
</head>
```

### Option 2: NPM distribution

In `angular.json`, add the JS file to `scripts` and the WASM file to `assets`:

```json
{
  "projects": {
    "your-app": {
      "architect": {
        "build": {
          "options": {
            "scripts": [
              "node_modules/@rdkit/rdkit/dist/RDKit_minimal.js"
            ],
            "assets": [
              {
                "glob": "RDKit_minimal.wasm",
                "input": "node_modules/@rdkit/rdkit/dist/",
                "output": "/"
              }
            ]
          }
        }
      }
    }
  }
}
```

### RDKit loader service

Use an Angular service with `ReplaySubject` to manage the RDKit module lifecycle:

```typescript
import { Injectable, OnDestroy } from "@angular/core";
import { Observable, ReplaySubject } from "rxjs";
import { first } from "rxjs/operators";

declare global {
  interface Window {
    initRDKitModule: any;
  }
}

@Injectable({ providedIn: "root" })
export class RDKitLoaderService implements OnDestroy {
  private rdkitSubject$!: ReplaySubject<any>;

  ngOnDestroy(): void {
    this.rdkitSubject$.complete();
  }

  getRDKit(): Observable<any> {
    if (!this.rdkitSubject$) {
      this.rdkitSubject$ = new ReplaySubject(1);
      window.initRDKitModule().then(
        (instance: any) => this.rdkitSubject$.next(instance),
        (error: any) => this.rdkitSubject$.error(error)
      );
    }
    return this.rdkitSubject$.asObservable().pipe(first());
  }
}
```

### Canvas renderer component

```typescript
import { Component, Input, OnInit, AfterViewInit, OnDestroy } from "@angular/core";
import { RDKitLoaderService } from "../rdkit-loader/rdkit-loader.service";

@Component({
  selector: "app-canvas-renderer",
  template: `<canvas [id]="id" [width]="width" [height]="height"></canvas>`
})
export class CanvasRendererComponent implements OnInit, OnDestroy {
  @Input() id!: string;
  @Input() structure!: string;
  @Input() subStructure: string = "";
  @Input() width: number = 250;
  @Input() height: number = 200;

  private rdkit: any;
  private sub: any;

  constructor(private rdkitService: RDKitLoaderService) {}

  ngOnInit() {
    this.sub = this.rdkitService.getRDKit().subscribe((rdkit: any) => {
      this.rdkit = rdkit;
      this.draw();
    });
  }

  draw() {
    const mol = this.rdkit.get_mol(this.structure || "invalid");
    if (!mol) return;

    const canvas = document.getElementById(this.id) as HTMLCanvasElement;

    if (this.subStructure) {
      const qmol = this.rdkit.get_qmol(this.subStructure);
      if (qmol) {
        const details = mol.get_substruct_match(qmol);
        mol.draw_to_canvas_with_highlights(canvas, details);
        qmol.delete();
      }
    } else {
      mol.draw_to_canvas(canvas, -1, -1);
    }

    mol.delete();
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
  }
}
```

### SVG renderer component

```typescript
import { Component, Input, OnInit, OnDestroy } from "@angular/core";
import { RDKitLoaderService } from "../rdkit-loader/rdkit-loader.service";
import { DomSanitizer, SafeHtml } from "@angular/platform-browser";

@Component({
  selector: "app-svg-renderer",
  template: `<div [innerHTML]="svgContent"></div>`
})
export class SvgRendererComponent implements OnInit, OnDestroy {
  @Input() id!: string;
  @Input() structure!: string;
  @Input() subStructure: string = "";
  @Input() width: number = 250;
  @Input() height: number = 200;

  svgContent!: SafeHtml;
  private rdkit: any;
  private sub: any;

  constructor(
    private rdkitService: RDKitLoaderService,
    private sanitizer: DomSanitizer
  ) {}

  ngOnInit() {
    this.sub = this.rdkitService.getRDKit().subscribe((rdkit: any) => {
      this.rdkit = rdkit;
      this.draw();
    });
  }

  draw() {
    const mol = this.rdkit.get_mol(this.structure || "invalid");
    if (!mol) return;

    let svg: string;
    if (this.subStructure) {
      const qmol = this.rdkit.get_qmol(this.subStructure);
      const details = mol.get_substruct_match(qmol);
      svg = mol.get_svg_with_highlights(details);
      qmol?.delete();
    } else {
      svg = mol.get_svg();
    }

    this.svgContent = this.sanitizer.bypassSecurityTrustHtml(svg);
    mol.delete();
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
  }
}
```

### Usage in templates

```html
<!-- Canvas -->
<app-canvas-renderer
  id="mol1"
  structure="CC(=O)Oc1ccccc1C(=O)O"
  [width]="300"
  [height]="250"
></app-canvas-renderer>

<!-- Substructure highlight -->
<app-canvas-renderer
  id="mol2"
  structure="CC(=O)Oc1ccccc1C(=O)O"
  subStructure="c1ccccc1"
  [width]="300"
  [height]="250"
></app-canvas-renderer>

<!-- SVG -->
<app-svg-renderer
  id="mol3"
  structure="CC(=O)Oc1ccccc1C(=O)O"
  [width]="300"
  [height]="250"
></app-svg-renderer>
```



