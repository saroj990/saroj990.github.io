+++
title = 'Tree Shaking in Component Libraries: What Actually Works'
date = '2026-07-21T12:00:00+05:30'
draft = false
description = 'A granular guide to tree shaking for component libraries — ESM, sideEffects, barrel files, bundler behavior, and how to verify unused components really drop out of the bundle.'
tags = ['JavaScript', 'Tree Shaking', 'Component Libraries', 'Bundlers', 'Frontend']
categories = ['Engineering', 'Frontend']
summary = 'Tree shaking only works when libraries ship analyzable ESM, declare side effects correctly, and avoid barrel/CJS traps. Here is how it works in practice.'
+++

<img src="/images/posts/tree-shaking/cover.svg" alt="Tree shaking keeps only imported components in the bundle" width="960" height="400" />

*You import one button. The library still contains a date picker, a chart kit, and a rich-text editor. Tree shaking is the mechanism that should keep those unused pieces out of the final bundle.*

Component libraries are where tree shaking either pays off or quietly fails. A design system might ship hundreds of modules. Your app may use twenty. If shaking works, users download less JavaScript. If it does not, every route pays for components nobody rendered.

This post goes deep on how tree shaking actually works for component libraries — from ESM and `sideEffects`, to barrel files, bundler differences, authoring pitfalls, and how to prove unused code is gone.

**TL;DR:** Tree shaking needs **static ESM**, **correct side-effect metadata**, and a **production minify/DCE pass**. Most “my library does not tree shake” bugs come from CJS output, CSS/polyfill side effects, or barrel files that pull the whole graph into the bundle.

---

## What Tree Shaking Means (Precisely)

**Tree shaking** is a form of dead-code elimination aimed at **unused module exports**: the bundler builds a dependency graph from `import` / `export`, marks what is reachable from your entry points, and drops exports that are never used.

It is not magic compression. It is **reachability analysis** plus **minifier-powered unused-code removal**.

Important distinctions:

| Term | Meaning |
|------|---------|
| **Tree shaking** | Remove unused **module exports** based on static imports |
| **Dead code elimination (DCE)** | Broader: remove code that can never run (including inside functions) |
| **Minification** | Rename/compress; often enables deeper DCE |
| **Code splitting** | Lazy-load chunks; complementary, not the same thing |

For component libraries, the practical question is:

> If I `import { Button } from '@acme/ui'`, does `Modal`, `DataGrid`, and `WYSIWYG` still end up in my production JS?

---

## The Three Conditions Bundlers Need

<img src="/images/posts/tree-shaking/requirements.svg" alt="ESM, side effects metadata, and production DCE" width="960" height="360" />

*If any of these is missing, unused components often survive.*

### 1. ES modules with static imports/exports

Tree shaking depends on **static structure**:

```js
// Analyzable
import { Button } from '@acme/ui'
export function App() {
  return <Button>Save</Button>
}
```

These are hard or impossible to shake reliably:

```js
// Dynamic — bundler cannot prove what is needed
const name = condition ? 'Button' : 'Modal'
const mod = await import(`@acme/ui/${name}`)

// CommonJS — generally not tree-shakeable in the ESM sense
const ui = require('@acme/ui')
ui.Button
```

Why ESM matters:

- `import` / `export` are static and hoistable  
- `require()` can be conditional and computed  
- Bundlers can mark live bindings and unused exports in ESM graphs  

If your library’s published entry resolves to **CJS only**, consumers usually will not get effective shaking of named component exports. Some bundlers can still drop unreachable code inside a CJS file after minify, but that is not the same as reliably dropping entire unused sibling components from a design-system entry.

### 2. Side-effect information

Even with ESM, a module might do work at import time:

```js
// side effect: runs on import, even if you never use Button
import './button.css'
console.log('Button loaded')
registerPolyfill()
```

Bundlers must assume such modules are needed unless told otherwise.

In `package.json`, Webpack popularized (and other tools now honor):

```json
{
  "sideEffects": false
}
```

Meaning: “files in this package are safe to drop if none of their exports are used.”

Or be precise:

```json
{
  "sideEffects": [
    "**/*.css",
    "**/*.scss",
    "./src/polyfills.js"
  ]
}
```

Meaning: those paths have side effects; other modules are considered pure for shaking purposes.

Rollup/Vite production builds also benefit from honest side-effect metadata at package boundaries, but Webpack is especially sensitive to this field.

**Component-library implication:** if every component file starts with `import './styles.css'` and your package says `"sideEffects": false`, you can break styling in production. If you omit `sideEffects` entirely, Webpack assumes files may have side effects and may keep more modules than you expect.

### 3. A production optimize/minify pass

Development builds often keep more code for better stack traces and faster rebuilds. Unused-export removal is strongest when:

- `mode: 'production'` (Webpack)  
- minify enabled (Terser / esbuild / SWC)  
- library code is ESM so unused exports can be dropped before/during minify  

Always measure **production** bundles, not `next dev` / `vite` dev graphs.

---

## How a Component Library Is Usually Consumed

### Named import from a barrel

```js
import { Button, Stack } from '@acme/ui'
```

This is the DX everyone wants. It only shakes well if the library entry is ESM and the barrel does not introduce unavoidable side effects.

### Deep / path import

```js
import { Button } from '@acme/ui/button'
// or
import Button from '@acme/ui/Button'
```

This bypasses a mega-barrel. Many mature libraries document this pattern because it is more predictable across bundlers and older toolchains.

### Namespace import (usually worse)

```js
import * as UI from '@acme/ui'
return <UI.Button />
```

Some bundlers can still shake this, but it is easier to accidentally retain more exports. Prefer named imports.

---

## Barrel Files: The #1 Library Footgun

A **barrel file** re-exports many modules from one entry:

```js
// @acme/ui/index.js
export * from './button'
export * from './modal'
export * from './table'
export * from './charts'
```

Barrels are convenient. They are also where tree shaking goes to die when combined with:

- CSS imported in the barrel or in every child  
- CJS/`export *` interop edge cases  
- “clever” runtime registration in the index  
- bundlers that are conservative about `export *`  

<img src="/images/posts/tree-shaking/barrel-risk.svg" alt="Risky barrel versus shake-friendly library design" width="960" height="380" />

*Barrels are fine only when the graph stays pure and ESM-clean.*

### What goes wrong

Suppose `charts.js` contains a heavy charting dependency and also:

```js
import './charts.css'
Chart.registerGlobalPlugins() // side effect
export function Chart() { /* ... */ }
```

If importing `@acme/ui` evaluates that module (or the bundler cannot prove it is unused), your “Button-only” page ships chart code.

### Mitigations used in the wild

1. **Ship per-component entry points** via `package.json` `"exports"`  
2. **Keep barrels pure**: only `export` lines, no CSS, no registration  
3. **Mark side-effect files explicitly** in `sideEffects`  
4. **Avoid `export *` from CJS wrappers**  
5. **App-side compile helpers** (historically `babel-plugin-import`; in Next.js, `optimizePackageImports`)  

Next.js can rewrite barrel imports for known packages:

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    optimizePackageImports: ['@acme/ui', 'lucide-react'],
  },
}

module.exports = nextConfig
```

This compile-time rewrite turns `import { Button } from '@acme/ui'` into a more granular import so the bundler does not have to trust a mega-barrel.

---

## Authoring a Shake-Friendly Component Library

### Publish ESM (and be honest in package.json)

Modern shape:

```json
{
  "name": "@acme/ui",
  "type": "module",
  "sideEffects": ["**/*.css"],
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./button": {
      "types": "./dist/button.d.ts",
      "import": "./dist/button.js"
    },
    "./package.json": "./package.json"
  }
}
```

Notes:

- Prefer **conditional exports** over a single legacy `"main"` pointing at CJS.  
- If you must ship CJS for older consumers, give ESM via `"import"` and accept that CJS consumers get weaker shaking.  
- Do not point `"module"` / `"import"` at code that still contains `require()`.

### Keep modules small and one-component-focused

Good:

```text
button/index.js        -> Button only
modal/index.js         -> Modal only
date-picker/index.js   -> DatePicker + its local helpers
```

Risky:

```text
components.js          -> every component in one file
index.js               -> imports all CSS + registers everything
```

### Treat CSS as a declared side effect

Options libraries use:

1. **CSS files imported by components** + `"sideEffects": ["**/*.css"]`  
2. **CSS-in-JS runtime** (bundle cost is different; still not “free”)  
3. **Zero-runtime CSS** (Linaria, compiled vanilla-extract, etc.)  
4. Document that consumers must import `@acme/ui/styles.css` once  

There is no universal best option — but lying with `"sideEffects": false` while importing CSS is a classic production bug.

### Avoid top-level work in component modules

Bad:

```js
analytics.track('button-module-evaluated')
polyfillResizeObserver()
export function Button() {}
```

Good:

```js
export function Button() {
  // pure render path; side effects inside effects if needed
}
```

If you need registration (custom elements, chart plugins), put it behind an explicit `register()` export or a deep import the consumer opts into.

### Preserve purity through the compiler

If you transpile with Babel/TS:

- For libraries, compile to **ESM** (`modules: false` in Babel, or `module: "ESNext"` / bundler module settings appropriately).  
- Compiling library source to CJS before publish often destroys shaking for consumers.  
- Prefer Rollup/tsup/unbuild/vite-lib mode that emits ESM with clean bindings.

Terser/esbuild understand `/*#__PURE__*/` annotations for calls that are safe to drop if unused:

```js
export const Button = /*#__PURE__*/ forwardRef(...)
```

This helps when helpers wrap component definitions.

---

## Consumer-Side Checklist (App Developers)

Even a perfect library can be defeated by app config.

1. **Import only what you use** — named imports, avoid `import *`  
2. **Prefer documented deep imports** when the library recommends them  
3. **Build production** before measuring  
4. **Do not transpile `node_modules` to CJS** unnecessarily  
5. **Alias to ESM entry** if a package ships dual builds and your resolver picks CJS  
6. **Analyze the bundle** — do not trust feelings  

### How to verify shaking worked

Use a visualizer:

- `rollup-plugin-visualizer`  
- Webpack Bundle Analyzer  
- `source-map-explorer`  
- Vite’s build analyze plugins  

Then search the production chunk for a string that exists only in an unused component (a unique error message, SVG path, or dependency).

If you imported only `Button` but see `DatePicker` unique strings in the main chunk, shaking failed.

Also compare sizes:

```bash
# example with an analyzer in your toolchain
npm run build && npm run analyze
```

---

## Worked Mini-Example

### Library layout

```text
@acme/ui/
  package.json
  dist/
    index.js          (barrel re-exports)
    button.js
    modal.js
    heavy-chart.js
```

`button.js`:

```js
export function Button(props) {
  return <button {...props} />
}
```

`heavy-chart.js`:

```js
import * as Heavy from 'some-huge-chart-vendor'
export function HeavyChart(props) {
  return <Heavy.Chart {...props} />
}
```

`index.js`:

```js
export { Button } from './button.js'
export { Modal } from './modal.js'
export { HeavyChart } from './heavy-chart.js'
```

`package.json`:

```json
{
  "type": "module",
  "sideEffects": false,
  "exports": {
    ".": "./dist/index.js",
    "./button": "./dist/button.js",
    "./heavy-chart": "./dist/heavy-chart.js"
  }
}
```

### App usage

```js
import { Button } from '@acme/ui'
```

**Expected (healthy ESM + sideEffects false):** `HeavyChart` and `some-huge-chart-vendor` do not appear in the app’s production bundle.

**Fails if:**

- published entry is CJS  
- barrel imports CSS for all components  
- `heavy-chart.js` is imported for side effects from `index.js`  
- app does `import * as UI from '@acme/ui'` and references `UI` in a way that retains exports  

Safer explicit import when in doubt:

```js
import { Button } from '@acme/ui/button'
```

---

## Bundler Notes (Practical, Not Mythical)

| Bundler | Notes for component libs |
|---------|--------------------------|
| **Rollup** | Designed around ESM shaking; excellent for libraries themselves |
| **Webpack 5** | Good ESM shaking; respects `sideEffects`; barrels can still be tricky |
| **Vite (Rollup/esbuild)** | Strong in production builds; still depends on package entry quality |
| **esbuild** | Fast DCE/minify; tree shaking exists but package boundary details matter |
| **Next.js** | Use `optimizePackageImports` for barrel-heavy packages (often under `experimental`, depending on Next version) |

Do not assume “Vite means free tree shaking regardless of package shape.” The package must still expose shakeable ESM.

---

## Common Myths

**“Named imports always tree shake.”**  
Only if the resolved module graph is ESM and free of blocking side effects.

**“`sideEffects: false` is always correct.”**  
Wrong if you have CSS, polyfills, or ambient registration. Be precise.

**“Deep imports are outdated.”**  
They are still the most portable escape hatch across bundlers and monorepos.

**“If TypeScript compiles, shaking works.”**  
`tsc` emitting CJS `require` for your library publish will hurt consumers.

**“Devtools network size proves shaking.”**  
Measure production output, ideally with source maps and an analyzer.

---

## A Design Checklist for Library Teams

Before you tag a release as “tree-shakeable,” confirm:

1. Production consumer build uses your **ESM** export condition  
2. `sideEffects` is either `false` or an explicit list that matches reality  
3. Entry barrels do not execute registration/CSS for unused components  
4. Heavy optional features are **separate entry points** (`@acme/ui/charts`)  
5. CI builds a sample app that imports one component and **fails if heavy deps appear**  
6. Docs show both barrel and deep-import forms  
7. You tested at least one of Webpack + Vite/Rollup, because app ecosystems differ  

An automated smoke test is worth more than a README claim.

---

## How This Fits Real Component Libraries

Patterns you will recognize:

- **Icon libraries** — thousands of exports; barrel imports without optimization are notorious for huge bundles  
- **Date pickers / charts / editors** — should usually be optional entry points  
- **Design tokens + CSS** — side effects that must be declared or loaded intentionally  
- **Monorepo packages** — easy to accidentally publish mixed CJS/ESM graphs  

If you maintain an internal design system, treat “Button-only app bundle excludes DataGrid vendor code” as a **release gate**, not a hope.

---

## Conclusion

Tree shaking in component libraries is less about a bundler checkbox and more about **package architecture**:

> ESM exports + honest side-effect metadata + small modules + optional heavy entry points + production verification

Get those right, and `import { Button } from '@acme/ui'` can mean what developers think it means. Get them wrong, and every page ships half the design system.

**Next steps for your own library:** add an `exports` map, set precise `sideEffects`, split one heavy component into its own entry, and add a CI bundle assertion that fails when unused vendor code reappears.
