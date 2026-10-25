# `tsdown-plugin-shallow` plugin

## Summary

Shallow bundling bundles the code of one chosen dependency into a library's output, but keeps that dependency's own dependencies external. The external imports left in the output are merged into a newly generated publish package's `package.json`, so a consumer who installs the library also gets everything the bundled code imports. The plugin does not modify the root manifest. Existing publish files are copied or symlinked into the generated package at their original relative paths.

```ts
// tsdown.config.ts
import { defineConfig } from 'tsdown'
import { shallow } from 'tsdown-plugin-shallow'

export default defineConfig({
  plugins: [shallow({ include: ['foo'], packageDir: '.shallow/package' })]
})
```

## Motivation

A library author sometimes wants to ship a dependency's code inside their own package instead of depending on it. Common reasons:

- `foo` is an internal workspace package that is never published.
- `foo` is published, but the author wants to pin an exact build of it, or patch it, without forcing that version on consumers.
- `foo` is small, and depending on it would pull in a package the author doesn't want consumers to see.

tsdown can already do this: put `foo` in `devDependencies` and import it, and tsdown bundles it. But tsdown then bundles everything `foo` imports too, and everything those packages import, all the way down. If `foo` depends on `zod`, the library now ships its own copy of `zod`. That has real costs:

- **Duplication.** A consumer who also uses `zod` gets two copies, and cannot dedupe them.
- **Broken identity.** `instanceof` checks, symbols and module-level state no longer match between the two copies.
- **No updates.** Security fixes in `zod` don't reach the bundled copy until the library is rebuilt and republished.
- **Size and licenses.** The output grows, and the author now redistributes third-party code.

What the author usually wants is narrower: bundle `foo` itself, and treat `foo`'s dependencies as if the library declared them.

### Why tsdown bundles the whole tree today

tsdown's deps plugin (`src/features/deps.ts` in tsdown) keeps an import external only when its package name appears in the **root** `package.json`'s `dependencies`, `peerDependencies` or `optionalDependencies`. It never looks at which module is importing. When code inside `foo` imports `zod`, tsdown checks the root manifest, doesn't find `zod`, and bundles it.

The fix is to make that decision depend on the importer: an import made from inside `foo`, for a package that `foo` declares, stays external.

## Terminology

- **Root package**: the package being built. Its `package.json` is the author-maintained source for generation.
- **Publish package**: the generated package directory containing a new manifest and copies or symlinks of the root's publish files.
- **Shallow package**: a package matched by `include`. Its own code is bundled.
- **Hoisted dependency**: a dependency of a shallow package that is kept external and must therefore be declared by the publish package.

## API

```ts
import type { TsdownPlugin } from 'tsdown'

export interface ShallowOptions {
  /**
   * Packages to bundle shallowly, matched by package name.
   */
  include: string | RegExp | Array<string | RegExp>

  /**
   * Which hoisted dependencies are merged into the generated `package.json`.
   * - `'imported'`: only those the output actually imports, after tree-shaking.
   * - `'declared'`: every dependency the shallow package declares.
   * @default 'imported'
   */
  collect?: 'imported' | 'declared'

  /**
   * Directory for the generated publish package, relative to the root package.
   * Must be separate from the root and the build output directories.
   */
  packageDir: string

  /**
   * How existing publish files are placed in the generated package.
   * @default 'copy'
   */
  packageFiles?: 'copy' | 'symlink'

  /**
   * What to do when the ranges the root and shallow packages need for the same dependency have no intersection.
   * @default 'error'
   */
  onConflict?: 'error' | 'warn'
}

export function shallow(options: ShallowOptions): TsdownPlugin
```

The shallow package itself normally stays in the root's `devDependencies`. Its code is in `dist/`, so consumers don't need it installed.

## Example

The library depends on `foo` only at build time:

```jsonc
// package.json (root, unchanged)
{
  "dependencies": { "picocolors": "^1.1.0" },
  "devDependencies": { "foo": "^2.0.0" }
}
```

```jsonc
// node_modules/foo/package.json
{
  "dependencies": { "zod": "^4.1.0", "picocolors": "^1.0.0" },
  "peerDependencies": { "typescript": ">=5.0.0" }
}
```

After `tsdown` runs with the configuration above, `dist/index.mjs` contains `foo`'s code and still has `import { z } from 'zod'`. The generated manifest is:

```jsonc
// .shallow/package/package.json (generated)
{
  "dependencies": { "picocolors": "^1.1.0", "zod": "^4.1.0" },
  "peerDependencies": { "typescript": ">=5.0.0" },
  "devDependencies": { "foo": "^2.0.0" }
}
```

- `zod` is added with the range `foo` declares.
- `picocolors` is kept as written, because the root's `^1.1.0` lies inside `foo`'s `^1.0.0`, so it is already the intersection of the two.
- `typescript` is added as a peer, because `foo` expects its host to provide it.
- `foo` stays a dev dependency.
- The root manifest is unchanged; additions and range intersections apply only to the generated manifest.

The generated package preserves the original layout:

```text
.shallow/package/
  package.json       # generated from the root manifest
  dist/              # copied or symlinked from the root's dist/
  README.md          # copied or symlinked, when present
  LICENSE            # copied or symlinked, when present
```

For example, an existing `exports` target of `./dist/index.mjs` remains unchanged. Publish the generated package, not the root package.

## Detailed design

### 1. Working with tsdown options

- Hooks the user set in tsdown's config keep working.
- If the user set `deps.onlyBundle` to a list, shallow packages are added to it, so they are not reported as unexpected bundled dependencies. If `deps.onlyBundle` is unset, it stays unset, because setting it would turn tsdown's hint into a strict allowlist.

### 2. Finding shallow packages

A shallow package is identified by its **root directory**, not just by its name, because one name can be installed more than once:

- pnpm stores it at `node_modules/.pnpm/foo@2.0.0/node_modules/foo`.
- A workspace package is a symlink, and is resolved to its real path, for example `packages/foo`.
- A nested copy can sit at `node_modules/bar/node_modules/foo`.

The root of a shallow package is the nearest directory above the resolved import whose `package.json` has a matching `name`. Every installed copy is tracked separately, with its own manifest.

A module belongs to a shallow package when its path is inside that package's root directory, and no `/node_modules/` segment comes after the root. Without that second check, a package nested inside `foo/node_modules` would be treated as part of `foo`.

### 3. Keeping dependencies external

Imports made from outside shallow packages are left to tsdown as usual. An import made from inside `foo` is decided before tsdown's own rules apply, by its package name (`@scope/pkg/sub` gives `@scope/pkg`):

1. Relative and absolute paths, virtual modules and `data:` URLs are `foo`'s own files, and are bundled.
2. Node built-in modules are left to tsdown.
3. A package that matches `include` is bundled. This is another shallow package, or `foo` importing itself through a subpath.
4. A package that `foo` declares in `dependencies`, `optionalDependencies` or `peerDependencies` stays external, and is hoisted with the range `foo` declares.
5. Any other package is one `foo` never declared. It works in development only because the package manager hoisted it. This is an error, because there is no version range to write.

Declaration files follow the same rules. When a `.d.ts` file inside `foo` imports `bar`, and `foo` declares `@types/bar` in `dependencies`, then `@types/bar` is hoisted too. If `foo` lists `@types/bar` only in `devDependencies`, its published types are already broken for its own consumers. The plugin warns, but doesn't hoist it.

### 4. Which hoisted dependencies are collected

With `collect: 'imported'`, a dependency hoisted in step 3 may still be removed by tree-shaking. Only the hoisted dependencies that the final output still imports, in JS and declaration files alike, are collected.

With `collect: 'declared'`, every dependency the shallow package declares is hoisted, whether the output imports it or not. This suits shallow packages that load dependencies in ways the bundler can't see, such as `require.resolve` or a plugin loader.

### 5. Generating the publish package

A build can produce several formats. Generation runs once after every format has finished and after tsdown's own manifest processing. Each build starts from the current root manifest, never from the previous generated manifest. The plugin only writes inside `packageDir`; it does not change the root manifest or workspace configuration. Any root manifest updates made by tsdown itself remain controlled by tsdown's configuration.

The generated manifest copies the root's fields and merges hoisted dependencies. Apart from these dependency changes, only publishing-related configuration may need adjustment. Fields such as `name`, `version`, `exports`, `main`, `module`, `types`, `bin`, `files`, and unrelated scripts are preserved as written. The exact treatment of publishing configuration and lifecycle scripts is an unresolved question below.

Existing build output and other publish files are copied or symlinked into `packageDir`, preserving their paths relative to the root. This includes files such as README and LICENSE when present. File selection should follow the package's publish-file rules rather than copying the whole source checkout. The generated directory itself must be excluded from selection to avoid recursive inclusion. Copying is the default; symlink mode must meet the same final tarball contents requirement.

The destination is generated output and should be ignored by version control. Regeneration must also remove obsolete generated files so an earlier build cannot leave stale files in the published package. It must never delete or modify the source files behind a symlink.

#### Dependency merging

A hoisted dependency goes into the field that matches how the shallow package declares it:

| Shallow package declares | Publish package field                        |
| ------------------------ | -------------------------------------------- |
| `dependencies`           | `dependencies`                               |
| `optionalDependencies`   | `optionalDependencies`                       |
| `peerDependencies`       | `peerDependencies`                           |
| `peerDependenciesMeta`   | `peerDependenciesMeta`, copied with the peer |

If the root already declares the dependency, the two ranges are merged by intersecting them. A version range is a set of versions, and the intersection is exactly the versions that both the root and `foo` accept:

- **One range lies inside the other.** The narrower range is kept as written. For example, the root has `^1.2.0` and `foo` needs `^1.0.0`, so the generated manifest keeps `^1.2.0`. If instead the root has `^1.0.0` and `foo` needs `^1.2.0`, the generated manifest uses `^1.2.0`, because a consumer could otherwise install `1.0.0`, which breaks `foo`.
- **The ranges partly overlap.** The intersection is written as a new range. For example, the root has `>=1.0.0 <1.5.0` and `foo` needs `^1.2.0`, so the generated manifest uses `>=1.2.0 <1.5.0`.
- **The intersection is empty.** For example, the root has `^2.0.0` and `foo` needs `^1.0.0`. Nothing in `package.json` can fix this: `foo` would get the root's copy. This is a conflict, handled by `onConflict`. The way out is to bundle that dependency too, or not to bundle `foo` shallowly.
- **The root has it only in `devDependencies`.** The output now imports it at runtime, so it is added to `dependencies` with the required range. The dev entry is left alone.
- **The root has it in `peerDependencies`, and `foo` has it in `dependencies`.** The root's peer already guarantees that it is installed, so it stays a peer, with the intersection as its range.

Generation logs each added dependency and each range narrowed in the publish manifest, together with the shallow package that requires it. None of these changes are written back to the root.

When two shallow packages need the same dependency, their ranges are intersected the same way, and an empty intersection is a conflict.

Ranges that aren't semver, such as `workspace:`, `catalog:`, `npm:` aliases, `file:`, `link:`, git or URL ranges, are not compared. They are copied only when the root doesn't declare the dependency yet, and only `workspace:` and `catalog:` are allowed. Publishing must run with the workspace context needed for the package manager to replace those protocols with real versions; the plugin does not resolve them itself. Anything else is an error, because it would not install from the registry.

The generated manifest keeps the source file's indentation and trailing newline, and inserts new keys in alphabetical order. It is written only when its content changes. Local builds and CI use the same generation behavior; there are no root-manifest `'write'` or `'check'` modes.

#### Workspace and publishing guidance

After generation, the plugin prints the publish package's path and a reminder to include that directory in the workspace configuration when using workspace-aware installation or publishing, especially with `workspace:` or `catalog:` dependencies. It does not edit workspace configuration automatically. For example:

```text
Publish package generated at .shallow/package.
Include this directory in your workspace configuration when using workspace publishing.
Run package checks and publish against this generated package.
```

The workspace layout must account for the generated package retaining the root's name. The plugin must not silently rename either package; guidance for workspace tools that reject duplicate package names remains to be settled.

Publishing and CI should target the generated package. Existing build scripts and unrelated manifest fields are not rewritten to arrange this automatically.

### 6. Dependencies that are no longer needed

Each generated manifest starts from the root manifest and the current build's collected dependencies. A dependency added only by a previous build disappears when it is no longer collected. A previously narrowed range returns to the root's declared range when the additional constraint disappears. No ownership record is needed because the previous generated manifest is never used as input.

Dependencies explicitly declared in the root are always preserved. After the merge, the plugin warns about ordinary dependencies that no output file imports, but does not remove author-maintained entries. Such a dependency may be used outside the bundle, for example by a `bin` script that is copied into the publish package.

## Alternatives considered

- **Updating the root manifest.** This mixes author-maintained declarations with generated dependency requirements. Later builds cannot safely remove generated entries or restore ranges that were narrowed without tracking previous values and manual edits. Generating a fresh publish package avoids that state.
- **Flattening build output into the publish package root.** This would require rewriting `exports`, `main`, `types`, and other paths. Preserving the existing directory layout keeps those fields unchanged.
- **Bundle everything** (tsdown's default for dev dependencies). This is the behavior this RFC avoids, for the reasons in [Motivation](#motivation).
- **`bundleDependencies` in `package.json`.** npm then ships `foo` with its whole `node_modules` tree in the tarball. That duplicates even more than bundling, and pnpm handles it poorly.
- **A hand-written `deps.neverBundle` function.** Rolldown passes the importer to `external`, so an author can keep `foo`'s imports external by hand. But they must also prepare the publish manifest by hand, with no version checks. This plugin handles both the import decisions and the generated package.
- **Publishing `foo`.** Often the right answer, but not possible for private workspace packages, and not what the author wants when the goal is to hide or pin `foo`.

## Unresolved questions

1. **Checking the generated package.** tsdown's existing `publint` and `attw` checks run before generation and do not validate the final publish package. How should those checks target the generated directory after all formats finish?
2. **Publishing configuration and hooks.** Which publishing-related fields need adjustment in the generated manifest? In particular, directory redirection and publish lifecycle scripts must not redirect away from the generated package or recursively rebuild it. Unrelated fields remain unchanged; the exact publishing policy needs to be specified before implementation.
3. **Publish-file selection and symlinks.** Which existing packing API should select the files, and how should symlinks be materialized if a package manager does not include their targets? Both modes must produce a self-contained tarball with the same contents, including README, LICENSE, and any files needed by `bin`.
4. **Workspace integration.** How should authors include the generated package while keeping the original name, avoiding duplicate-name conflicts, and retaining the context needed for `workspace:` and `catalog:` conversion? The plugin should offer actionable guidance without editing workspace configuration.
5. **Depth.** Should `include` allow `foo>bar`, meaning bundle `bar` only when `foo` imports it? It resolves conflicts in [Generating the publish package](#5-generating-the-publish-package) without bundling `bar` everywhere, but it adds a lot of complexity, and it could be added later without breaking changes.
