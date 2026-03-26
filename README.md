# Knip bug: workspace dependencies reported as unused in single-workspace mode

## The issue

In a monorepo, when a package imports a **sibling workspace package**, knip incorrectly reports that dependency as unused — but only when knip is invoked from within the package directory (single-workspace mode).

Running from the monorepo root, or using `--workspace` from the root, correctly recognizes the dependency as used.

## Reproduction

```sh
pnpm install

# From root — CORRECT: no issues reported
pnpm knip --include dependencies

# From root with workspace filter — CORRECT: no issues reported
pnpm knip --include dependencies --workspace package-a

# From package directory — BUG: @repo/package-b IS reported unused
cd package-a
pnpm knip --include dependencies
# Unused dependencies (1)
# @repo/package-b  package.json:6:6
```

## Why it matters to me (and maybe others too)

This behavior can be important in an extremely large monorepo where scanning all the packages on each change is extremely expensive. Task orchestrators like `lage` and `turborepo` can be smart about which packages need to have `knip` run on them in a CI job based on whether there were changes. But this can only work if single-workspace mode works roughly the same way. (This kind of setup sacrifices some features, like `--include-entry-exports`, but the tradeoff can be worth it for the performance gain in extremely large repos.)

---

## AI-generated root-cause analysis and suggested fix

*Note: While I've carefully reviewed and refined the reproduction of the issue above, I haven't yet done the same for the root cause analysis below. I'm still including it in case it's helpful.*

### Root cause

The bug is in `packages/knip/src/graph/build.ts`, in the `analyzeSourceFile` callback (~line 400).

#### How dependency tracking works

When knip analyzes a source file, it resolves each import specifier via TypeScript's module resolver. For a workspace dependency like `@repo/package-b`, the resolver follows the **symlink** in `node_modules` and resolves to the real path: `package-b/index.js`.

Because this resolved path is **not inside `node_modules`**, TypeScript sets `isExternalLibraryImport = false`. This causes `_getImportsAndExports` to classify the import as **internal** (not external). The dependency never enters the `external` set at the source analysis level.

To compensate, `build.ts` has a recovery loop (lines 400-410) that scans `file.imports.imports` for internal imports that are actually workspace packages:

```typescript
for (const _import of file.imports.imports) {
  if (_import.filePath) {
    const packageName = getPackageNameFromModuleSpecifier(_import.specifier);
    if (packageName && isInternalWorkspace(packageName)) {
      file.imports.external.add({ ..._import, specifier: packageName });
      // ...
    }
  }
}
```

#### Where it breaks

`isInternalWorkspace` is defined as:

```typescript
const isInternalWorkspace = (packageName) =>
  chief.availableWorkspacePkgNames.has(packageName);
```

When knip runs from the **root** with all workspaces configured, `availableWorkspacePkgNames` contains both `@repo/package-a` and `@repo/package-b`. The check passes and the dependency is correctly promoted to `external`.

When knip runs from **within `package-a`** (single-workspace mode), `availableWorkspacePkgNames` contains only `@repo/package-a`. The sibling package `@repo/package-b` is not recognized as an internal workspace. The `if` branch is skipped, and there is no `else` branch to catch this case.

The import is:
- Not in `external` (because `isExternalLibraryImport` was false)
- Not in `unresolved` (because the module resolved successfully)
- Skipped by the `isInternalWorkspace` recovery check

It silently falls through. Nothing marks the dependency as referenced. Knip reports it as unused.

#### The full data flow

```
index.js
  import { add } from '@repo/package-b'
    │
    ├─ TypeScript resolves to: package-b/index.js
    │  (not in node_modules → isExternalLibraryImport = false)
    │
    ├─ _getImportsAndExports:
    │    internal: YES (added to internal map and imports set)
    │    external: NO  (not added — isExternalLibraryImport is false)
    │
    └─ build.ts recovery loop:
         packageName = "@repo/package-b"
         isInternalWorkspace("@repo/package-b") = false  ← BUG
         │
         └─ Falls through. Dependency never marked as referenced.
```

### Suggested fix

Add an `else if` branch to the recovery loop in `build.ts` that catches the case where a package-name-like specifier resolves outside `node_modules` but is not a known workspace:

```typescript
for (const _import of file.imports.imports) {
  if (_import.filePath) {
    const packageName = getPackageNameFromModuleSpecifier(_import.specifier);
    if (packageName && isInternalWorkspace(packageName)) {
      file.imports.external.add({ ..._import, specifier: packageName });
      if (!isGitIgnored(_import.filePath)) {
        pp.addProgramPath(_import.filePath);
      }
    } else if (packageName && isStartsLikePackageName(packageName) && !isInNodeModules(_import.filePath)) {
      // In single-workspace mode, sibling workspace packages resolve via
      // symlinks to paths outside node_modules. They won't appear in
      // availableWorkspacePkgNames, so the check above misses them.
      // Treat them as external so the dependency is marked as referenced.
      file.imports.external.add({ ..._import, specifier: packageName });
    }
  }
}
```

The conditions are:
1. `packageName` exists — the specifier looks like a package name (scoped or unscoped)
2. `isStartsLikePackageName(packageName)` — extra guard that it's a valid npm-style name
3. `!isInNodeModules(_import.filePath)` — the resolved file is outside `node_modules`, meaning it reached the real path via a symlink (the hallmark of a workspace dependency)

Both `isStartsLikePackageName` and `isInNodeModules` are already imported in `build.ts`.

This fix does not add the file as a program path (unlike the `isInternalWorkspace` branch), since in single-workspace mode we don't want to crawl into the sibling workspace's source tree. We only need to mark the dependency as referenced.
