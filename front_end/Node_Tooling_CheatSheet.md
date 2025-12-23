# Node Tooling Cheat Sheet: nvm, npm, npx, pnpm

## What each tool is for

**Node.js**: the runtime that executes JavaScript outside the browser.

**nvm (Node Version Manager)**: installs and switches Node.js versions per machine/user/project needs.

**npm (Node Package Manager)**: installs and manages project dependencies and runs scripts.

**npx**: runs a package/tool without needing a global install (often used for scaffolding).

**pnpm ("performant npm")**: an alternative package manager like npm, optimized for speed + disk efficiency.

## How they relate (mental model)

- nvm chooses the Node.js version
- node runs tools/apps
- npm / pnpm install dependencies + run project scripts
- npx / pnpm dlx run "one-off" CLIs (scaffolders, generators)

## Key differences

### npm vs pnpm

Both manage package.json dependencies and run scripts.

- pnpm uses a shared global store + linking → typically faster and uses less disk space
- pnpm is stricter about dependency resolution (helps catch missing deps)
- Lockfiles differ: `package-lock.json` (npm) vs `pnpm-lock.yaml` (pnpm)

### npm vs npx

- npm installs/manages packages
- npx runs a package's executable (often temporarily), e.g. project generators

### npx vs pnpm dlx

npx (npm ecosystem) ≈ pnpm dlx (pnpm ecosystem) for running one-off tools.

## How to choose (simple rules)

- Use nvm if you need to switch Node versions across projects
- Use one package manager per repo (don't mix lockfiles)

### Pick based on repo defaults:

- `package-lock.json` → npm
- `pnpm-lock.yaml` → pnpm

### If starting fresh:

- npm = simplest default
- pnpm = great if you care about speed/disk or use monorepos
