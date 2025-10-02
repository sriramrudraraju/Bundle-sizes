### Project initial setup

```tsx
// src/index.ts

export function greet(name: string = "World"): string {
  const PREFIX: string = "Hello";
  return `${PREFIX}, ${name}!`;
}
```

```json
// package.json
{
  ...,
  "type": "module",
  "scripts": {
    "build:esm": "esbuild src/index.ts --bundle --format=esm --platform=browser --outfile=dist/esm/bundle.js --minify --sourcemap --metafile=dist/esm/meta.json",
    "build:cjs": "esbuild src/index.ts --bundle --format=cjs --platform=browser --outfile=dist/cjs/bundle.js --minify --sourcemap --metafile=dist/cjs/meta.json",
    "visualize:esm": "esbuild-visualizer --metadata dist/esm/meta.json --filename dist/esm/visualizer.html",
    "visualize:cjs": "esbuild-visualizer --metadata dist/cjs/meta.json --filename dist/cjs/visualizer.html",
    "build": "concurrently --kill-others 'npm:build:esm' 'npm:build:cjs'",
    "visualize": "concurrently --kill-others 'npm:visualize:esm' 'npm:visualize:cjs'"
  },
  "devDependencies": {
    "concurrently": "^9.2.1",
    "esbuild": "^0.23.0",
    "esbuild-visualizer": "^0.7.0",
    "typescript": "^5.5.0"
  },
  ...
}
```

When checking the output of `npm run build`, we can see that the ESM bundle is smaller than the CJS bundle.

```
dist/esm/bundle.js       98b
dist/cjs/bundle.js       552b

// these are file sizes, not tree-shaken sizes
```

Q) Why is the ESM bundle smaller?
A) CommonJS needs extra wrapper/runtime code in the browser; ESM doesn’t.
CJS for the browser is not native. esbuild must add helpers to emulate CommonJS semantics (e.g., module/exports wiring, interop helpers like **commonJS/**toESM). Even for a trivial file, that wrapper overhead dominates, so dist/cjs/bundle.js is bigger.
Tree-shaking is generally more effective and simpler with ESM. With CJS, esbuild has to preserve more structure because require can be dynamic at runtime. Even though your current code is simple, you still pay for the wrapper.

Lets check for node platform

```json
// package.json

{
  ...,

  "scripts": {
    "build:esm": "... --platform=node ...",
    "build:cjs": "... --platform=node ...",
    ...
  },
  ...
}
```

now the sizes are:

```
dist/esm/bundle.js       98b
dist/cjs/bundle.js       580b

// these are file sizes, not tree-shaken sizes
```

Q) WHAAAATTTT!!!! CJS is even bigger than previous with platform=node!!!!
A) Changing --platform to node doesn’t remove the CommonJS bundling overhead. The size gap is driven by the CJS format’s wrapper/interop code, not just the platform.
--platform=node mainly affects polyfills and target assumptions, not the CJS wrapper itself:
For browser CJS, esbuild adds shims to emulate require/exports in the browser.
For node CJS, esbuild still needs the bundler wrapper to convert your ESM source to CJS and to stitch modules together when bundling. That wrapper remains even on Node.

If you must keep a CJS bundle, try

- Turn off sourcemaps for the size check:
  Remove --sourcemap from build:cjs when you’re measuring.
- Consider not bundling CJS for Node:
  If this is for Node consumption, try not to use any bundlers. Just use tsc to emit plain CJS. The bundler wrapper is what costs bytes; no bundle means almost no wrapper.

import { Button } from "@backyard/react";
For many UI libraries, the top-level barrel re-exports every component and often eagerly imports their modules (and sometimes CSS), which prevents aggressive tree-shaking. If the package is CommonJS at the top-level, or marks itself as having side effects, esbuild will conservatively include most of it.

Why esbuild might include “all components”
• Barrel file imports everything. The top-level @backyard/react may import all components internally before re-exporting. That pulls them all in.
• CommonJS export on the entry. If the package’s entry is CJS, esbuild’s tree-shaking is limited and it may keep entire modules.
• Package marks side effects. If @backyard/react has sideEffects: true (or no sideEffects), esbuild won’t drop unused top-level code.
• CSS side effects. If the barrel imports global CSS from all components, those imports are side-effectful and will be retained.
• Missing prod defines. Without process.env.NODE_ENV defined as production, dev-only branches and warnings may remain in the bundle.
Your current build scripts in
package.json
use esbuild correctly for a baseline, but they target --platform=node and don’t define process.env.NODE_ENV. Also, since you import from the barrel, esbuild likely can’t prune much.

1. node_modules pacakes' have sideEffects:false . Why is it not considered?
2. You said, treeshaking is not doable on CJS barrel files.. I get that.. But my code using import syntax and it should import from ESM syntax of npm package right?

Here’s what’s going on:

1. Why isn’t sideEffects:false being considered?
   esbuild ignores package.json sideEffects. That hint is a webpack-specific optimization. esbuild relies on static ESM analysis (symbol-level tree shaking) and top-level side-effect analysis instead of the package-level “sideEffects” flag.
   CJS defeats esbuild’s tree-shaking. If the entry you resolve to is CommonJS (e.g., a barrel that does require() of every component), esbuild will conservatively keep those requires because CJS is not statically analyzable like ESM. In that case, “sideEffects:false” cannot help.
   Barrel side effects keep modules. If the top-level entry/barrel imports CSS or runs top-level code, those imports are inherently side-effectful and won’t be dropped even when exports are unused.
   You’re importing the barrel:
   src/index.tsx
   imports @backyard/react. If that top-level entry aggregates all components, esbuild has little to prune regardless of sideEffects flags.
2. “I’m using ESM import syntax; why doesn’t it pick the ESM build?”
   Using import in your code doesn’t guarantee that esbuild resolves the dependency to ESM. Resolution is controlled by the package’s exports map (or main/module fields) and the active “conditions,” which esbuild derives from your build options.
   Your scripts use --platform=node (see
   package.json
   ), so the "node" condition is active. Many packages map "node" to a CJS build even when they also ship ESM for "browser"/"default".
   Example pattern in dependencies:
   exports: { ".": { "node": "./cjs/index.cjs", "default": "./esm/index.js" } }
   With --platform=node, the "node" condition wins → CJS.
   “import syntax” is not enough. esbuild prefers "import" over "require" when both are simultaneously available for the same condition, but if the package first branches on "node" vs "browser" and the "node" branch points to CJS, you still get CJS.
   When an exports map exists, mainFields preference doesn’t apply. So even if you try --main-fields=module,main, the exports conditions still decide which file you get.

Here’s the beginner-friendly summary with concrete evidence from your project.

What decides which file is imported from @backyard/react?

Import syntax in your code doesn’t pick the file by itself.
Writing import { Button } from '@backyard/react' just says “I want this package.” It doesn’t force ESM or CJS.
esbuild chooses which file to load based on the package’s fields and your build “platform.”
In node_modules/@backyard/react/package.json we found:
"main": "./index.js" ← CommonJS barrel (CJS)
"module": "./esm/index.js" ← ES module (ESM)
"exports": undefined ← no conditional exports
"sideEffects": false
Because there is no "exports", esbuild falls back to “main fields.” The default order depends on platform:
platform=node → prefer "main" over "module" (so CJS wins)
platform=browser → prefer "browser", then "module", then "main" (so ESM often wins)
Your build is using --platform=node.
In package.json you run esbuild with --platform=node.
That makes esbuild select "main" (CJS) instead of "module" (ESM).
Proof from your bundle

dist/esm/meta.json and dist/cjs/meta.json both include node_modules/@backyard/react/index.js (the CJS barrel).
That CJS barrel re-exports every component by requiring all component modules, which is why “everything” shows up.
With CJS, tree-shaking is limited; esbuild can’t safely drop those requires.
So the short answer to your question

Even though @backyard/react provides both "module" and "main", and even though your code uses import syntax, the resolver still follows esbuild’s platform-specific rules.
With --platform=node, esbuild prefers "main" (CJS) and you end up bundling the CommonJS barrel that pulls in all components.
How to make it pick ESM and only Button

Prefer ESM via platform or mainFields:
Switch to --platform=browser when measuring browser bundle size, or
Keep node platform but add --main-fields=module,main to prefer ESM.

Q)is the above theroy same for all othe bundlers like webpack and vite as well?
A) Yes, the core idea is the same across bundlers (webpack, Vite/Rollup, esbuild): what actually gets loaded from a package is determined by the package’s export fields and the bundler’s active “conditions” (platform/target), not by your import syntax alone. There are some differences in defaults and features, but the high‑level behavior is consistent.

How this maps to each bundler

Webpack 5
Resolution
Honors package.json exports when present. The active conditions depend on target (e.g., target: "web" vs "node"). If an exports map branches on "node" vs "browser", webpack picks the branch that matches the target first. Only then do "import" vs "require" sub-conditions apply.
If there’s no exports, it falls back to main fields. Default order depends on target:
target: "web" → ["browser", "module", "main"]
target: "node" → ["main", "module"]
So, using import syntax doesn’t force ESM if the chosen branch points to CJS.
Tree-shaking
Best with ESM. Limited for CJS.
Webpack DOES use package.json sideEffects to drop unused modules. sideEffects: false helps webpack prune unused files; but if the entry is a CJS barrel that requires many submodules, those requires are still executed/retained.
Practical takeaway
If target: "node", webpack is likely to prefer "main" (often CJS). If you want ESM, either target the browser, adjust resolve.mainFields, or import an ESM subpath.
Vite (Rollup under the hood)
Resolution
Honors exports. Client builds assume browser conditions; SSR builds use node conditions. If the exports map routes "node" to CJS, SSR can pick CJS even with import syntax.
Without exports, Rollup tends to prefer "module" over "main" for browser builds. Import syntax still doesn’t override higher-priority conditions.
Tree-shaking
Excellent for ESM; limited for CJS (same as others).
The package.json sideEffects flag is not used by default. Rollup/Vite rely on ESM’s static analysis and a treeshake.moduleSideEffects setting if you want to get aggressive. CSS imports are inherently side-effectful.
Practical takeaway
In client builds Vite usually picks ESM, but for SSR it can pick CJS depending on the package’s exports map. Subpath ESM imports are still the most reliable way to avoid the barrel.
esbuild (what you’re using)
Resolution
Honors exports (when present) and otherwise uses main fields. Platform sets which conditions win (browser vs node). Import syntax doesn’t force ESM if the chosen condition points to CJS.
Tree-shaking
Great for ESM; limited for CJS.
Ignores package.json sideEffects. It relies on static analysis of top-level side effects instead.
What this means for you

The “platform/target chooses the branch first, then import vs require” rule applies across webpack, Vite/Rollup, and esbuild.
If a package routes the node condition to a CJS barrel, all three can end up pulling that CJS barrel—even if you wrote ESM import syntax—unless you:
Build for the browser (so the browser/default branch with ESM is chosen), or
Override main fields (webpack/esbuild), or
Import a documented ESM subpath directly (e.g., @backyard/react/esm/Button), avoiding the barrel altogether.

Q) explain me how these imports work.. Will it work on backyard/react exports in package.json? or will it work based on my projects setup? and how will my setup influence that?

```
import { Button } from "@backyard/react";
```

A) import { Button } from "@backyard/react"; only identifies the package. It does not force ESM or CJS.
Where the actual file comes from

esbuild (and other bundlers) look into the package’s
package.json
in node_modules/@backyard/react/ and choose an entry based on package fields and your build settings:
If exports exists, it wins. The chosen file depends on active “conditions” (e.g., node vs browser, import vs require).
If exports is absent (your case), esbuild falls back to “main fields.”
What we found in @backyard/react/package.json

exports: not present
main: ./index.js (CommonJS barrel)
module: ./esm/index.js (ESM)
sideEffects: false
How your setup influences resolution

In
package.json
, your build uses --platform=node. With esbuild, the default main-field order is:
platform=node → prefer "main" then "module" → you get CJS (index.js)
platform=browser → prefer "browser" then "module" then "main" → you typically get ESM
Proof in your build outputs:
Both dist/esm/meta.json and dist/cjs/meta.json reference node_modules/@backyard/react/index.js (the CJS barrel), which re-exports every component and drags them all in.
Important nuances:
Your code’s import syntax (ESM) doesn’t override that choice.
esbuild’s --format=esm|cjs only controls your bundle’s output format, not which dependency file is picked.
tsconfig.json
options (e.g., module, moduleResolution) apply to TypeScript type-checking, not esbuild’s package resolution.
sideEffects: false in the package helps webpack; esbuild relies on static ESM analysis and top-level side-effect analysis, so it doesn’t use that flag.

Q) what happens in backyard/react only has ESM code? How will the resolution works?
A) The bundler (esbuild/webpack/Vite) picks the target from exports using active conditions. If only ESM is provided, it will select the ESM file.
Your project’s output format doesn’t block this:
If you output ESM (--format=esm), it stays ESM.
If you output CJS (--format=cjs), esbuild will transpile the ESM dependency into CJS inside your bundle.
If you mark the package as external (not bundled), then your runtime must support ESM when loading it. For example, a CJS Node app cannot require() an ESM-only package; you would need dynamic import() or type: "module".
