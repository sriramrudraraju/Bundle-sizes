# Import Resolution, tsconfig, and esbuild: A Beginner’s Field Guide (with a smile)

Ever written `import { Component } from "some-library"` and wondered which file you actually got? You’re not alone. Imports go on a little adventure before they land in your bundle. This guide explains that trip. How resolution works, what `tsconfig` does (and doesn’t), and how esbuild (and other bundlers) bundles everything.

---

## TL;DR

- **Import syntax doesn’t force ESM output.** `import ... from "pkg"` doesn’t guarantee ESM output; the package and bundler decide.
- **"exports" in package.json beats everything.** If present, `"exports"` controls which files and subpaths are public.
- **Platform matters.** `--platform=node` often leads to CJS; `--platform=browser` often leads to ESM.
- **Deep imports point at files.** `pkg/subpath` resolves directly to files (absolute file in that path) unless blocked by `"exports"`.
- **Tree-shaking loves ESM.** Barrels and CSS side effects defeat it.
- **tsconfig is for types.** It doesn’t steer bundler resolution.

---

## Table of Contents

1. What does what?
2. The package “menu”: exports, main, module, browser
3. Platform and conditions: why target changes the file
4. Deep imports vs. bare imports
5. Tree-shaking 101 (and barrels)
6. sideEffects: webpack vs. esbuild
7. tsconfig: what it does (and doesn’t)
8. esbuild flags that matter
9. Forcing ESM (two reliable strategies)
10. Dynamic import and CJS/ESM caveats
11. /_ @**PURE** _/ annotations
12. A tiny measurement harness
13. Examples
14. FAQ
15. Handy checklist
16. References (official docs)

---

## 1) What does what?

- **[Your import]** You request a package or subpath, e.g. `import { X } from "widgets-ui"`.
- **[Resolver]** Figures out the exact file using Node‑style rules, package fields, and conditions.
- **[Bundler (esbuild)]** Parses (CJS/ESM), tree‑shakes, minifies, and emits your requested output format.
- **[TypeScript]** Provides types and checks; it doesn’t control bundler resolution.

---

## 2) The package “menu”: exports, main, module, browser

When there’s no `exports`, bundlers fall back to “main fields”:

```json
{
  "name": "widgets-ui",
  "main": "./cjs/index.js",
  "module": "./esm/index.js",
  "types": "./index.d.ts"
}
```

With exports, it’s the boss:

```json
{
  "name": "widgets-ui",
  "exports": {
    ".": {
      "import": "./esm/index.js",
      "require": "./cjs/index.cjs",
      "default": "./esm/index.js",
      "types": "./index.d.ts"
    },
    "./button": "./esm/button/index.js"
  }
}
```

[exports rules] Only listed subpaths are public ("." and "./button"). Others are blocked.

## 3) Platform and conditions: why target changes the file

- [When no exports] in package.json defined, esbuild uses main fields.
- browser prefers "module".
- node prefers "main" first; if missing, tries "module".
- [With exports] the active conditions (e.g. import, require) pick the target.
- [Key point] Writing import in your code doesn’t force the "import" target.

## 4) Deep imports vs. bare imports

- [Bare import] import { Button } from "widgets-ui" → bundler picks entry using exports/main fields and conditions.
- [Deep import] import { Button } from "widgets-ui/button" → resolves directly to files/folders (node_modules/widgets-ui/button/index.js), unless blocked by "exports".
- [Encapsulation] If "exports" is defined and doesn’t include "./button", deep import must fail.

## 5) Tree‑shaking 101 (and barrels)

- [ESM is static] Tree‑shaking works best with ESM import/export.
- [CJS is dynamic] Limited tree‑shaking for require()/module.exports.
- [Barrels] Files that import and re‑export everything often drag in the world.
- [CSS side effects] Global CSS imports are side‑effectful; they won’t be dropped.

## 6) sideEffects: webpack vs. esbuild

- [webpack] Uses "sideEffects": false to drop unused files.
- [esbuild] Ignores package‑level sideEffects; relies on static analysis of ESM and top‑level effects.

## 7) tsconfig: what it does (and doesn’t)

- [Does] Types, editor tooling, TS compilation behavior.
- [Doesn’t] Control esbuild’s package resolution or which file is chosen at bundle time.

## 8) esbuild flags that matter

- [Prefer ESM] --main-fields=module,main
- [Browser builds] --platform=browser
- [Drop dev branches] --define:process.env.NODE_ENV="production"
- [Externalize peers] --external:react --external:react-dom
- [Inspect bundle] --metafile=dist/meta.json
  Example scripts:

```json
{
  "scripts": {
    "build:esm": "esbuild src/index.ts --bundle --format=esm --platform=browser --main-fields=module,main --define:process.env.NODE_ENV=\"production\" --external:react --external:react-dom --outfile=dist/esm/bundle.js --minify --sourcemap --metafile=dist/esm/meta.json",
    "build:cjs": "esbuild src/index.ts --bundle --format=cjs --platform=browser --main-fields=module,main --define:process.env.NODE_ENV=\"production\" --external:react --external:react-dom --outfile=dist/cjs/bundle.js --minify --sourcemap --metafile=dist/cjs/meta.json"
  }
}
```

## 9) Forcing ESM (two reliable strategies)

[Deep import ESM subpath]

```tsx
import { Button } from "widgets-ui/esm/button";
```

Bypasses a CJS barrel and improves tree‑shaking.
Caveat: If the package later adds exports but doesn’t include "./esm/button", this can break.

[Keep root import, prefer ESM]

```bash
esbuild src/index.ts --bundle --platform=browser --main-fields=module,main ...
```

## 10) Dynamic import and CJS/ESM caveats

[CJS can’t require ESM synchronously]
A CJS app can’t require() an ESM‑only package; use dynamic import() or "type": "module".
[Code splitting] Dynamic import() enables splitting but adds async boundaries.

## 11) /_ @**PURE** _/ annotations

[Pure calls can be dropped] if their results are unused:

```js
/* @__PURE__ */ makeWidget({ color: "blue" });
```

esbuild understands this and removes dead pure calls when minifying.

## 13) Examples

[Bare import (root)]

```tsx
import { Button } from "widgets-ui";
// Outcome depends on platform/conditions and package fields
```

[Deep import (path exists, no exports blocking)]

```tsx
import { Button } from "widgets-ui/button";
// Resolves to node_modules/widgets-ui/button/index.js
```

[Deep ESM import (explicit ESM subpath)]

```ts
import { Button } from "widgets-ui/esm/button";
```

[Simple app function with types]

```ts
export function greet(name: string = "World"): string {
  const PREFIX: string = "Hello";
  return `${PREFIX}, ${name}!`;
}
```

[Conditional exports example (library‑side)]

```json
{
  "exports": {
    ".": {
      "import": "./esm/index.js",
      "require": "./cjs/index.cjs",
      "default": "./esm/index.js"
    },
    "./button": "./esm/button/index.js"
  }
}
```

## 14) FAQ

- [Why did I still get CJS with import syntax?]
  Your build’s conditions or main‑field order pointed to CJS. Code syntax doesn’t overrule package resolution.
- [Does tsconfig control bundling?]
  No. It’s for TypeScript, not the bundler’s resolver.
- [Why is my bundle huge?]
  CJS barrels, CSS side effects, and not preferring ESM. Use ESM subpaths, set --main-fields=module,main, define production.
- [Is "sideEffects": false helping?]
  Helps webpack; esbuild ignores it and uses static analysis.

## 15) Handy checklist

- [Prefer ESM] --platform=browser, --main-fields=module,main
- [Avoid barrels] Import ESM subpaths when supported
- [Drop dev code] --define:process.env.NODE_ENV="production"
- [Keep peers external] --external:<peer>
- [Measure] --metafile, gzip/brotli sizes
- [Mind side effects] Especially global CSS

## 16) References (official docs)

    [esbuild: conditions] https://esbuild.github.io/api/#conditions
    [esbuild: platform] https://esbuild.github.io/api/#platform
    [esbuild: main-fields] https://esbuild.github.io/api/#main-fields
    [esbuild: packages] https://esbuild.github.io/api/#packages
    [esbuild FAQ: tree-shaking] https://esbuild.github.io/faq/#tree-shaking
    [Node.js: conditional exports] https://nodejs.org/api/packages.html#conditional-exports
