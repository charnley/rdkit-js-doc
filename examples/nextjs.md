---
layout: page
title: Using RDKit.js with Next.js
menu: Next.js
permalink: /examples/nextjs/
---

RDKit.js loads a `.wasm` module asynchronously.
Turbopack (Next.js 16) needs `resolveAlias` to map the `.wasm` file.
The rdkit package also tries `require("fs")` for Node.js detection, and although never called in the browser, it complains in the static code analysis.

First, install an app

```sh
mkdir project && cd project
npx create-next-app@latest . --typescript --src-dir --app --no-tailwind --no-eslint
pnpm i @rdkit/rdkit
```

Configure `next.config.ts`.
`serverExternalPackages` tells Next.js to skip bundling `@rdkit/rdkit` on the server.
`resolveAlias` maps the `.wasm` file so Emscripten can find it in the browser.

The dummy is only for the browser bundle where fs doesn't exist.
And the `fs` alias is added and linked to a dummy function 

```ts
// ./next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  serverExternalPackages: ["@rdkit/rdkit"],
  turbopack: {
    resolveAlias: {
      "RDKit_minimal.wasm": "./node_modules/@rdkit/rdkit/dist/RDKit_minimal.wasm",
      fs: "./src/lib/dummy.ts",
    },
  },
};

export default nextConfig;
```

Where the dummy simply looks like

```ts
// src/lib/dummy.ts
export default {};
```

For the the frontend `"use client"` skips SSR.
`new URL("...", import.meta.url)` tells Turbopack to bundle the `.wasm`, then `locateFile` points at the bundled URL

```tsx
// src/app/rdkit-mol.tsx
"use client";

import { useEffect, useState } from "react";
import type { RDKitModule } from "@rdkit/rdkit";
import _initRDKitModule from "@rdkit/rdkit";

const initRDKitModule = _initRDKitModule as unknown as (
  options?: { locateFile?: () => string }
) => Promise<RDKitModule>;

const wasmUrl = new URL("RDKit_minimal.wasm", import.meta.url).href;

export default function RdkitMol() {
  const [RDKit, setRDKit] = useState<RDKitModule | null>(null);

  useEffect(() => {
    initRDKitModule({ locateFile: () => wasmUrl }).then(setRDKit);
  }, []);

  if (!RDKit) return <div>Loading RDKit...</div>;

  const mol = RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
  if (!mol) return <div>Invalid molecule</div>;

  const svg = mol.get_svg_with_highlights(
    JSON.stringify({
      width: 400,
      height: 300,
      bondLineWidth: 1,
      addStereoAnnotation: true,
    })
  );
  mol.delete();

  return <div dangerouslySetInnerHTML={{ __html: svg }} />;
}
```

Which can be used on a page

```tsx
// src/app/page.tsx
import RdkitMol from "./rdkit-mol";

export default function Home() {
  return (
    <main>
      <h1>RDKit.js + Next.js</h1>
      <RdkitMol />
    </main>
  );
}
```

## RDKit on server-side (API route)

With `serverExternalPackages`, Node.js runs `@rdkit/rdkit` natively, which finds the `.wasm` next to the `.js` file in `node_modules`:

```ts
// src/app/api/rdkit/route.ts
import { NextResponse } from "next/server";
import initRDKitModule from "@rdkit/rdkit"

export async function GET() {
  const RDKit = await initRDKitModule();

  const mol = RDKit.get_mol("CC(=O)Oc1ccccc1C(=O)O");
  const molblock = mol.get_molblock();
  mol.delete();

  return NextResponse.json({
    version: RDKit.version(),
    molblock,
  });
}
```

