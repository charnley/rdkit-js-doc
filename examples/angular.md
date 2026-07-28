---
layout: page
title: Using RDKit.js with Angular
menu: Angular
permalink: /examples/angular/
---


Angular's esbuild bundler needs two tricks:
copy the `.wasm` file to the build output, and tell esbuild to ignore a
Node-only import that emscripten leaves in the JS glue (dead code in browser).

```bash
npx @angular/cli new project_name --style=css --routing=false --skip-git --skip-tests --ssr=false
cd project_name
npm i @rdkit/rdkit
```

First configure `angular.json`.
Add the `.wasm` file to `assets` so Angular copies it to the build output.
Add `node:module` to `externalDependencies` so esbuild does not try to resolve the Node-only code inside `RDKit_minimal.js`.

`externalDependencies` tells esbuild: "do not try to resolve this import, leave it alone."
The browser then runs the file, skips the dead Node code, and the factory works.

```jsonc
// angular.json
{
  "projects": {
    "rdkit-angular": {
      "architect": {
        "build": {
          "options": {
            "assets": [
              { "glob": "**/*", "input": "public" },
              {
                "glob": "RDKit_minimal.wasm",
                "input": "node_modules/@rdkit/rdkit/dist",
                "output": "/"
              }
            ],
            "externalDependencies": ["node:module"]
          }
        }
      }
    }
  }
}
```

In practise it means we can write a small service wrapper class that loads the rdkit import.
You can then initialize rdkit with `initRDKitModule()`
which assumes root-level `.wasm` path, which is set but the above configuration.

```ts
// src/app/rdkit.service.ts
import { Injectable, signal } from '@angular/core';
import type { MainModule } from '@rdkit/rdkit';

@Injectable({ providedIn: 'root' })
export class RdkitService {
  private module: MainModule | null = null;
  readonly ready = signal(false);
  readonly error = signal<string | null>(null);

  async init(): Promise<MainModule> {
    if (this.module) return this.module;
    try {
      const factory = (await import('@rdkit/rdkit')).default;
      this.module = await factory({
        locateFile: (path: string) => `/${path}`,
      });
      this.ready.set(true);
      return this.module;
    } catch (err) {
      this.error.set(err instanceof Error ? err.message : String(err));
      throw err;
    }
  }

  get(): MainModule {
    if (!this.module) throw new Error('RDKit not initialized');
    return this.module;
  }
}
```

Which can then be used in a component like

```ts
// src/app/app.ts
import { Component, inject, signal, ViewEncapsulation } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';
import { RdkitService } from './rdkit.service';

@Component({
  selector: 'app-root',
  imports: [],
  templateUrl: './app.html',
  styleUrl: './app.css',
  encapsulation: ViewEncapsulation.None,
})
export class App {
  private readonly rdkit = inject(RdkitService);
  private readonly sanitizer = inject(DomSanitizer);

  protected readonly ready = this.rdkit.ready;
  protected readonly error = this.rdkit.error;
  protected readonly smiles = signal('CCO');
  protected readonly svg = signal<SafeHtml>('');
  protected readonly info = signal<string>('');

  constructor() {
    this.rdkit.init().then(() => this.render()).catch(() => {});
  }

  render(): void {
    if (!this.ready()) return;
    const module = this.rdkit.get();
    const mol = module.get_mol(this.smiles());
    if (!mol || !mol.is_valid()) {
      this.svg.set('');
      this.info.set('Invalid SMILES');
      mol?.delete();
      return;
    }
    this.svg.set(this.sanitizer.bypassSecurityTrustHtml(mol.get_svg(300, 300)));
    this.info.set(`SMILES: ${mol.get_smiles()}\nMolblock:\n${mol.get_molblock()}`);
    mol.delete();
  }

  onInput(event: Event): void {
    this.smiles.set((event.target as HTMLInputElement).value);
    this.render();
  }
}
```

```html
<!-- src/app/app.html -->
<main class="container">
  <h1>RDKit.js + Angular</h1>

  @if (error()) {
    <p class="error">Failed to load RDKit: {{ error() }}</p>
  } @else if (!ready()) {
    <p>Loading RDKit WASM…</p>
  } @else {
    <label>
      SMILES:
      <input type="text" [value]="smiles()" (input)="onInput($event)" />
    </label>

    @if (svg()) {
      <div class="drawing" [innerHTML]="svg()"></div>
    }

    @if (info()) {
      <pre>{{ info() }}</pre>
    }
  }
</main>
```
