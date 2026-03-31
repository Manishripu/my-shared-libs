# Angular Library Sharing via Git (Repo A → Repo B)
> A complete, production-ready guide to building, versioning, and consuming an Angular 19 library across repositories using Git — no npm registry required.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Phase 1 — Create the Library (Repo A)](#phase-1--create-the-library-repo-a)
4. [Phase 2 — Build the Library](#phase-2--build-the-library)
5. [Phase 3 — Configure for Git Installation](#phase-3--configure-for-git-installation)
6. [Phase 4 — Automate the Release Flow](#phase-4--automate-the-release-flow)
7. [Phase 5 — CI Validation](#phase-5--ci-validation)
8. [Phase 6 — Consume in Repo B](#phase-6--consume-in-repo-b)
9. [Phase 7 — Updating the Library](#phase-7--updating-the-library)
10. [Secondary Entrypoints (Advanced)](#secondary-entrypoints-advanced)
11. [Best Practices](#best-practices)
12. [Common Mistakes](#common-mistakes)
13. [Limitations](#limitations)
14. [Quick Reference Cheatsheet](#quick-reference-cheatsheet)

---

## Architecture Overview

```
Repo A (Angular Workspace)
  └── projects/common-lib/        ← Library source
        ↓
  ng build common-lib
        ↓
  dist/common-lib/                ← Compiled output (committed to Git)
        ↓
  git tag v1.0.0 + push
        ↓
Git Remote Repository
        ↓
  npm install git+https://...#v1.0.0
        ↓
Repo B (Consumer App)
  └── node_modules/@your-org/common-lib
```

---

## Prerequisites

| Tool | Minimum Version | Check Command |
|------|----------------|---------------|
| Node.js | 18+ | `node --version` |
| npm | 9+ | `npm --version` |
| Angular CLI | 19+ | `ng version` |
| Git | 2.30+ | `git --version` |

Install Angular CLI globally if needed:
```bash
npm install -g @angular/cli@19
```

---

## Phase 1 — Create the Library (Repo A)

### 1.1 — Generate the Angular Workspace

If you don't have one yet:
```bash
ng new repo-a --no-create-application
cd repo-a
git init
```

### 1.2 — Generate the Library

```bash
ng generate library common-lib
```

This creates the following structure:
```
repo-a/
├── angular.json
├── package.json                  ← Workspace root (do NOT use for lib metadata)
├── tsconfig.json
└── projects/
    └── common-lib/
        ├── ng-package.json       ← ng-packagr config
        ├── package.json          ← ✅ Library metadata lives here
        ├── tsconfig.lib.json
        ├── tsconfig.spec.json
        └── src/
            ├── public-api.ts     ← All exports go here
            └── lib/
                ├── common-lib.module.ts
                ├── common-lib.component.ts
                └── common-lib.service.ts
```

### 1.3 — Organise the Library Source

Expand the default structure for real-world use:
```
projects/common-lib/src/lib/
├── components/
│   ├── button/
│   │   ├── button.component.ts
│   │   ├── button.component.html
│   │   └── button.component.scss
│   └── modal/
│       └── ...
├── services/
│   ├── api.service.ts
│   └── auth.service.ts
├── models/
│   ├── user.model.ts
│   └── response.model.ts
├── utils/
│   ├── date.util.ts
│   └── string.util.ts
├── directives/
│   └── highlight.directive.ts
└── pipes/
    └── format.pipe.ts
```

### 1.4 — Configure the Public API

Edit `projects/common-lib/src/public-api.ts` — this is the single source of truth for what consumers can import:

```typescript
// Module
export * from './lib/common-lib.module';

// Components
export * from './lib/components/button/button.component';
export * from './lib/components/modal/modal.component';

// Services
export * from './lib/services/api.service';
export * from './lib/services/auth.service';

// Models
export * from './lib/models/user.model';
export * from './lib/models/response.model';

// Utils & Pipes (only export what consumers need)
export * from './lib/pipes/format.pipe';
```

> ⚠️ **Rule:** Only export what is part of the public contract. Internal helpers, abstract base classes, and test utilities should NOT be exported.

### 1.5 — Configure Library `package.json`

Edit `projects/common-lib/package.json`. This is separate from the workspace root and is the file that gets bundled into `dist/`:

```json
{
  "name": "@your-org/common-lib",
  "version": "1.0.0",
  "description": "Shared Angular component and service library",
  "keywords": ["angular", "library"],
  "license": "MIT",
  "peerDependencies": {
    "@angular/common": "^19.0.0",
    "@angular/core": "^19.0.0",
    "@angular/forms": "^19.0.0",
    "rxjs": "~7.8.0",
    "tslib": "^2.3.0"
  },
  "peerDependenciesMeta": {
    "@angular/forms": {
      "optional": true
    }
  },
  "dependencies": {}
}
```

> ✅ `ng-packagr` picks this file up and writes it (merged with build output paths) into `dist/common-lib/package.json` automatically. Do **not** add `main`, `module`, or `types` manually — the build tool sets them correctly.

---

## Phase 2 — Build the Library

### 2.1 — Run the Build

```bash
ng build common-lib
```

Output in `dist/common-lib/`:
```
dist/common-lib/
├── package.json          ← Auto-generated by ng-packagr
├── index.d.ts
├── esm2022/
│   └── common-lib.mjs
├── fesm2022/
│   └── common-lib.mjs    ← Primary ESM bundle
├── lib/
│   └── **/*.d.ts         ← Type declarations
└── README.md
```

### 2.2 — Build in Watch Mode (Development)

During active development, use watch mode to automatically rebuild on changes:
```bash
ng build common-lib --watch
```

Then in a separate terminal, run Repo B's app — it will pick up changes as they're rebuilt.

### 2.3 — Verify the Build Output

Quickly check that the build is correct:
```bash
# Verify package.json looks right
cat dist/common-lib/package.json

# Verify public types are exported
grep -r "export" dist/common-lib/index.d.ts
```

---

## Phase 3 — Configure for Git Installation

### 3.1 — Fix `.gitignore`

The workspace `.gitignore` likely has a blanket `dist/` ignore. Override it specifically for the library's compiled output. **Order matters:**

```gitignore
# .gitignore

# Ignore all dist output by default
/dist

# ✅ Exception: allow the compiled library (consumers install from this)
!/dist/common-lib
```

Verify the exception works:
```bash
git status dist/common-lib   # Should show as untracked (not ignored)
```

### 3.2 — Add `.npmignore` to Root (Optional but Recommended)

This prevents large workspace files from bloating the install in Repo B:
```
# .npmignore
src/
projects/
node_modules/
.github/
*.log
*.md
!README.md
angular.json
tsconfig*.json
karma.conf.js
```

---

## Phase 4 — Automate the Release Flow

Manual multi-step releases are error-prone. Replace them with a script.

### 4.1 — Install `release-it` (Recommended)

```bash
npm install --save-dev release-it @release-it/conventional-changelog
```

Create `.release-it.json` in the repo root:
```json
{
  "git": {
    "commitMessage": "chore: release v${version}",
    "tagName": "v${version}",
    "tagAnnotation": "Release v${version}"
  },
  "npm": {
    "publish": false
  },
  "plugins": {
    "@release-it/conventional-changelog": {
      "preset": "angular",
      "infile": "CHANGELOG.md"
    }
  },
  "hooks": {
    "before:init": "ng build common-lib",
    "after:bump": "git add dist/common-lib",
    "after:release": "echo ✅ Released v${version}"
  }
}
```

Add scripts to the **workspace root** `package.json`:
```json
{
  "scripts": {
    "build:lib": "ng build common-lib",
    "release": "release-it",
    "release:minor": "release-it minor",
    "release:major": "release-it major",
    "release:patch": "release-it patch"
  }
}
```

### 4.2 — Manual Release Flow (Without `release-it`)

If you prefer manual control, always follow this exact sequence:

```bash
# 1. Build first — NEVER tag before building
ng build common-lib

# 2. Bump version in projects/common-lib/package.json (manually or with npm version)
# Example: change "version": "1.0.0" to "version": "1.1.0"

# 3. Stage the compiled output AND package.json changes
git add dist/common-lib
git add projects/common-lib/package.json

# 4. Commit with a clear message
git commit -m "build: release v1.1.0"

# 5. Tag with an annotated tag (more reliable than lightweight tags)
git tag -a v1.1.0 -m "Release v1.1.0"

# 6. Push code AND tags together
git push origin main --follow-tags
```

> ⚠️ **Never use `git push origin --tags` alone** — push the branch commit alongside. A tag pointing to an unsynced commit confuses consumers.

### 4.3 — Commit Convention (for CHANGELOG)

Use [Conventional Commits](https://www.conventionalcommits.org/) so `release-it` can auto-generate a changelog:

| Prefix | Meaning | Version Bump |
|--------|---------|--------------|
| `feat:` | New feature | Minor |
| `fix:` | Bug fix | Patch |
| `perf:` | Performance improvement | Patch |
| `BREAKING CHANGE:` | API break | Major |
| `chore:` | Tooling, releases | None |
| `docs:` | Documentation only | None |

---

## Phase 5 — CI Validation

Add a GitHub Actions workflow to enforce that no broken build is ever tagged.

### 5.1 — Build & Test on Every Push

Create `.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build library
        run: ng build common-lib --configuration production

      - name: Run tests
        run: ng test common-lib --watch=false --browsers=ChromeHeadless

      - name: Lint
        run: ng lint common-lib
```

### 5.2 — Release Workflow (Tag-Triggered)

Create `.github/workflows/release.yml`:
```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build library
        run: ng build common-lib --configuration production

      - name: Verify dist was committed
        run: |
          if [ ! -d "dist/common-lib" ]; then
            echo "❌ dist/common-lib not found — did you commit the build?"
            exit 1
          fi

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
```

> ✅ This workflow verifies `dist/` was committed and creates a GitHub Release automatically on every version tag.

---

## Phase 6 — Consume in Repo B

### 6.1 — Install the Library

```bash
# Install using a specific version tag (recommended)
npm install git+https://github.com/your-org/repo-a.git#v1.0.0

# Or pin to an exact commit SHA (most reproducible)
npm install git+https://github.com/your-org/repo-a.git#abc1234ef
```

The resulting entry in Repo B's `package.json`:
```json
{
  "dependencies": {
    "@your-org/common-lib": "git+https://github.com/your-org/repo-a.git#v1.0.0"
  }
}
```

### 6.2 — Register in `tsconfig.json`

Add a path alias so TypeScript resolves imports correctly:
```json
{
  "compilerOptions": {
    "paths": {
      "@your-org/common-lib": ["node_modules/@your-org/common-lib"]
    }
  }
}
```

### 6.3 — Import the Module

In your `AppModule` or a feature module:
```typescript
import { NgModule } from '@angular/core';
import { CommonLibModule } from '@your-org/common-lib';

@NgModule({
  imports: [
    CommonLibModule
  ]
})
export class AppModule {}
```

For standalone components (Angular 14+):
```typescript
import { Component } from '@angular/core';
import { ButtonComponent } from '@your-org/common-lib';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ButtonComponent],
  template: `<lib-button>Click me</lib-button>`
})
export class AppComponent {}
```

### 6.4 — Use Services

```typescript
import { Component, inject } from '@angular/core';
import { ApiService } from '@your-org/common-lib';

@Component({ ... })
export class DashboardComponent {
  private api = inject(ApiService);

  ngOnInit() {
    this.api.getData().subscribe(data => console.log(data));
  }
}
```

### 6.5 — Verify Peer Dependencies

After install, ensure Angular versions align between Repo A and Repo B:
```bash
npm ls @angular/core   # Should show no conflicts
```

If you see version mismatches, either update Repo B or align `peerDependencies` in the library's `package.json`.

---

## Phase 7 — Updating the Library

### 7.1 — In Repo A: Release a New Version

```bash
# If using release-it:
npm run release:minor

# If manually:
ng build common-lib
# bump version in projects/common-lib/package.json
git add dist/common-lib projects/common-lib/package.json
git commit -m "build: release v1.1.0"
git tag -a v1.1.0 -m "Release v1.1.0"
git push origin main --follow-tags
```

### 7.2 — In Repo B: Update the Dependency

```bash
npm install git+https://github.com/your-org/repo-a.git#v1.1.0
```

> ⚠️ npm caches Git installs. If you're not seeing changes from a re-used tag, clear the cache:
> ```bash
> npm cache clean --force
> npm install
> ```

---

## Secondary Entrypoints (Advanced)

As your library grows, you'll want to support tree-shakeable imports like:
```typescript
import { formatDate } from '@your-org/common-lib/utils';
import { ApiService } from '@your-org/common-lib/services';
```

### Structure

```
projects/common-lib/
├── src/                   ← Primary entrypoint
│   └── public-api.ts
├── utils/                 ← Secondary entrypoint
│   ├── src/
│   │   ├── index.ts
│   │   └── date.util.ts
│   └── ng-package.json
└── services/              ← Secondary entrypoint
    ├── src/
    │   ├── index.ts
    │   └── api.service.ts
    └── ng-package.json
```

### Secondary `ng-package.json`

```json
{
  "$schema": "../../../node_modules/ng-packagr/ng-package.schema.json",
  "lib": {
    "entryFile": "src/index.ts"
  }
}
```

### Secondary `src/index.ts`

```typescript
export * from './date.util';
export * from './string.util';
```

> ✅ `ng build common-lib` will automatically discover and build secondary entrypoints. No extra config needed.

---

## Best Practices

### Version Management

- Always use **annotated tags** (`git tag -a v1.0.0 -m "..."`) not lightweight tags
- Never force-push tags — treat them as immutable
- For critical production consumers, **pin to a commit SHA** rather than a tag
- Maintain a `CHANGELOG.md` — consumers can't see your commit history easily

### Library Design

- Keep the library **framework-agnostic where possible** — avoid tight coupling to Angular-specific patterns in models/utils
- Use `peerDependencies` for Angular packages — never `dependencies`
- Never import from `@angular/core/testing` in library source
- Set `"sideEffects": false` in library `package.json` if applicable (enables better tree-shaking)

### Repository Hygiene

- Add a `README.md` to `projects/common-lib/` with API documentation
- Use consistent `exports` naming — avoid renaming things between major versions
- Run `ng build` and tests in CI before every tag push

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Exporting from `dist/` path in root `package.json` | Conflicts with workspace config | Use `projects/common-lib/package.json` only |
| Tagging before building | Consumers get stale/empty `dist/` | Always build → commit → tag (in that order) |
| Pinning to `main` branch | Non-deterministic installs, breaks on every commit | Always pin to a version tag or commit SHA |
| Not committing `dist/` | Install fails with empty package | Whitelist `dist/common-lib` in `.gitignore` |
| Adding `main`/`module`/`types` manually in root | Overrides ng-packagr output, breaks ESM | Let `ng-packagr` write these fields |
| Tight coupling to app-specific logic | Library becomes hard to reuse | Keep lib decoupled; use interfaces for dependencies |
| Using `dependencies` instead of `peerDependencies` | Installs duplicate Angular in Repo B | Always use `peerDependencies` for Angular packages |
| Missing exports in `public-api.ts` | `TS2305: Module has no exported member` | Export everything consumers need through `public-api.ts` |

---

## Limitations

| Limitation | Severity | Workaround |
|------------|----------|------------|
| No strict semantic versioning enforcement | Medium | Use `release-it` + conventional commits |
| Slower `npm install` vs registry | Low | Use `npm ci` with lockfile; first install is slow, subsequent are cached |
| Larger repo size (committed `dist/`) | Low | Acceptable for most teams; use shallow clones if needed |
| Tags are mutable (can be force-pushed) | High | Add branch protection rules; consumers can pin to SHA |
| No `npm audit` support for git deps | Medium | Review dependency tree manually on upgrades |
| Hard to use in monorepos with npm workspaces | Medium | Consider Nx or Turborepo if the team grows |

---

## Quick Reference Cheatsheet

### Repo A — Release

```bash
# Build
ng build common-lib

# Commit dist
git add dist/common-lib projects/common-lib/package.json
git commit -m "build: release v1.x.x"

# Tag + push
git tag -a v1.x.x -m "Release v1.x.x"
git push origin main --follow-tags
```

### Repo B — Install / Update

```bash
# Fresh install
npm install git+https://github.com/your-org/repo-a.git#v1.0.0

# Update to new version
npm install git+https://github.com/your-org/repo-a.git#v1.1.0

# Force re-install (clears npm cache)
npm cache clean --force && npm install
```

### Import Reference

```typescript
// NgModule-based
import { CommonLibModule } from '@your-org/common-lib';

// Standalone component
import { ButtonComponent } from '@your-org/common-lib';

// Service
import { ApiService } from '@your-org/common-lib';

// Secondary entrypoint (if configured)
import { formatDate } from '@your-org/common-lib/utils';
```

---

*Last updated: v1.1 — includes secondary entrypoints, CI workflows, release automation, and SHA pinning.*
