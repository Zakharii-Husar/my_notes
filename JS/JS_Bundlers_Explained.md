# JS Bundlers Explaine

A JS bundler takes an entry file, follows and resolves all imports into a dependency graph, transforms modules, and emits browser-ready optimized output (one or more files).





## Bundler Cheat Sheet

### 1. Import Resolution

Figures out what file/package each `import ... from "..."` actually points to (including `node_modules`, `package.json` exports, extensions, index files).

### 2. Transform / Transpile

Converts source formats into browser-ready code:

- **TypeScript/TSX/JSX** → JavaScript
- **SCSS** → CSS
- **Modern JS** → Target JS (ES5, ES2015, etc.)

### 3. Dependency Graph

Crawls from your entry file through every import to build a full "map" of everything your app uses.

### 4. Bundling / Output

Emits deployable files by stitching modules together (often into one or more JS "chunks") or by rewriting module paths for multi-file output.

### 5. Code Splitting

Produces multiple bundles so parts of the app load later/only when needed (commonly triggered by `dynamic import()` and route splitting).

### 6. Tree-shaking

Removes unused exports/code when it can prove they're not needed (works best with **ESM** + no side effects).

### 7. Asset Pipeline

Treats CSS/images/fonts as dependencies too: compiles, copies, fingerprints, and returns URLs so imports like `import logo from "./logo.png"` work.

### 8. Production Optimizations

- Minifies code
- Hashes filenames for caching
- Generates source maps for debugging

---

## What a Bundler is NOT (but often comes with)

### 1. Dev Server (Hot Reload)

Serves your app locally and pushes updates instantly (**HMR**) without full reloads; not bundling, just fast feedback.

### 2. Transpiler (Babel / SWC / esbuild)

A syntax/format converter; it can transform files but doesn't inherently do full graph bundling or chunking by itself.

### 3. Linter / Typecheck

Code quality + correctness checks (**ESLint**, **TypeScript** `tsc`) that don't produce build output; usually run alongside the build.

### 4. Test Runner

Executes tests (**Vitest**/Jest/Playwright/etc.); separate from producing your production bund


#JS
