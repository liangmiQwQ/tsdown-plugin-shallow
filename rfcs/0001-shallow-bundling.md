# RFC 0001: Shallow Bundling

## Summary

Shallow bundling bundles the code of one chosen dependency into a library's output, but keeps that dependency's own dependencies external. The external imports left in the output are then merged into the library's `package.json`, so a consumer who installs the library also gets everything the bundled code imports.

```ts
// tsdown.config.ts
import { defineConfig } from 'tsdown'
import { shallow } from 'tsdown-plugin-shallow'

export default defineConfig({
  plugins: [shallow({ include: ['foo'] })]
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

- **Root package**: the package being built, whose `package.json` the plugin may update.
- **Shallow package**: a package matched by `include`. Its own code is bundled.
- **Hoisted dependency**: a dependency of a shallow package that is kept external and must therefore be declared by the root package.

## API

```ts
import type { TsdownPlugin } from 'tsdown'

export interface ShallowOptions {
  /**
   * Packages to bundle shallowly, matched by package name.
   */
  include: string | RegExp | Array<string | RegExp>

  /**
   * Which hoisted dependencies are merged into the root `package.json`.
   * - `'imported'`: only those the output actually imports, after tree-shaking.
   * - `'declared'`: every dependency the shallow package declares.
   * @default 'imported'
   */
  collect?: 'imported' | 'declared'

  /**
   * What to do with the root `package.json`.
   * - `'write'`: merge hoisted dependencies into it.
   * - `'check'`: fail the build if it is out of date.
   * - `false`: only log what would change.
   * @default process.env.CI ? 'check' : 'write'
   */
  pkg?: 'write' | 'check' | false

  /**
   * What to do when the root and a shallow package need versions of the same dependency that can't both be met.
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
// package.json (root, before)
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

After `tsdown` runs with `shallow({ include: ['foo'] })`, `dist/index.mjs` contains `foo`'s code and still has `import { z } from 'zod'`. The root manifest becomes:

```jsonc
// package.json (root, after)
{
  "dependencies": { "picocolors": "^1.1.0", "zod": "^4.1.0" },
  "peerDependencies": { "typescript": ">=5.0.0" },
  "devDependencies": { "foo": "^2.0.0" }
}
```

- `zod` is added with the range `foo` declares.
- `picocolors` is kept as written, because the root's `^1.1.0` already satisfies `foo`'s `^1.0.0`.
- `typescript` is added as a peer, because `foo` expects its host to provide it.
- `foo` stays a dev dependency.

## Detailed design

### 1. Setup (`tsdownConfig` and `tsdownConfigResolved`)

The plugin uses tsdown's own plugin hooks, added in tsdown 0.21.9:

- `tsdownConfig` registers a `build:done` hook through `config.hooks`, wrapping any hooks the user already set, whether they are an object or a function.
- If the user set `deps.onlyBundle` to a list, `include` is appended to it, so the shallow package is not reported as an unexpected bundled dependency. If `deps.onlyBundle` is unset, it is left unset, because setting it would turn tsdown's hint into a strict allowlist.
- `tsdownConfigResolved` fires once per output format. The plugin stores the resolved `pkg` and `cwd`, and counts the formats per `package.json`, so it knows when the last format has finished.

### 2. Finding shallow packages

A shallow package is identified by its **root directory**, not just by its name, because one name can be installed more than once:

- pnpm stores it at `node_modules/.pnpm/foo@2.0.0/node_modules/foo`.
- A workspace package is a symlink, and Rolldown resolves it to its real path, for example `packages/foo`.
- A nested copy can sit at `node_modules/bar/node_modules/foo`.

When the plugin first sees an import that matches `include`, it resolves it, walks up from the resolved file to the nearest `package.json` whose `name` matches, and stores that directory with the parsed manifest. All copies are tracked.

A module belongs to a shallow package when its path is inside that package's root directory, and no `/node_modules/` segment comes after the root. Without that second check, a package nested inside `foo/node_modules` would be treated as part of `foo`.

### 3. Keeping dependencies external (`resolveId`)

The plugin registers `resolveId` with `order: 'pre'`. tsdown adds user plugins after its own deps plugin, and `order: 'pre'` makes this hook run first. For each import:

1. Skip if the importer doesn't belong to a shallow package.
2. Skip relative and absolute paths, virtual modules (`\0…`) and `data:` URLs. These are `foo`'s own files and are bundled.
3. Skip Node built-in modules. tsdown already handles them.
4. Parse the package name from the specifier (`@scope/pkg/sub` gives `@scope/pkg`).
5. If the name matches `include`, skip. Another shallow package, or `foo` importing itself through a subpath, is bundled.
6. If `foo` declares the name in `dependencies`, `optionalDependencies` or `peerDependencies`, return `{ id, external: true }` and record it as hoisted, together with the declaring package and its range.
7. Otherwise `foo` imports a package it never declared. It works in development only because the package manager hoisted it. The plugin reports an error, because it has no version range to write.

The same hook covers declaration files. tsdown's DTS build calls `resolveId` with `.d.ts` importers. When a `.d.ts` file inside `foo` imports `bar`, and `foo` declares `@types/bar` in `dependencies`, then `@types/bar` is hoisted too. If `foo` lists `@types/bar` only in `devDependencies`, its published types are already broken for its own consumers. The plugin warns, but doesn't hoist it.

### 4. Collecting what the output imports (`generateBundle`)

With `collect: 'imported'`, a dependency recorded in step 3 may still be removed by tree-shaking. In `generateBundle`, the plugin reads the external imports of every chunk (`chunk.imports` and `chunk.dynamicImports`), JS and declaration chunks alike, and keeps only the hoisted dependencies that appear there.

With `collect: 'declared'`, every dependency the shallow package declares is hoisted, whether the output imports it or not. This suits shallow packages that load dependencies in ways the bundler can't see, such as `require.resolve` or a plugin loader.

### 5. Merging into `package.json`

Each format builds separately, but there is only one `package.json`. Results are collected per format, and the merge runs once, in the `build:done` hook of the last format to finish. tsdown runs its own `package.json` step (`exports`, `inlinedDependencies`) before `build:done`, so the two writes never race.

A hoisted dependency goes into the field that matches how the shallow package declares it:

| Shallow package declares | Root field                                   |
| ------------------------ | -------------------------------------------- |
| `dependencies`           | `dependencies`                               |
| `optionalDependencies`   | `optionalDependencies`                       |
| `peerDependencies`       | `peerDependencies`                           |
| `peerDependenciesMeta`   | `peerDependenciesMeta`, copied with the peer |

If the root already declares the dependency:

- **Root range is a subset of the required range.** For example, the root has `^1.2.0` and `foo` needs `^1.0.0`. Keep the root range.
- **The ranges overlap, but the root range is wider.** For example, the root has `^1.0.0` and `foo` needs `^1.2.0`. A consumer could install `1.0.0`, which breaks `foo`. `'write'` narrows the root range to `foo`'s range and logs it. `'check'` fails.
- **The ranges don't overlap.** For example, the root has `^2.0.0` and `foo` needs `^1.0.0`. Nothing in `package.json` can fix this: `foo` would get the root's copy. This is a conflict, handled by `onConflict`. The way out is to bundle that dependency too, or not to bundle `foo` shallowly.
- **The root has it only in `devDependencies`.** The output now imports it at runtime, so it is added to `dependencies` with the required range. The dev entry is left alone.
- **The root has it in `peerDependencies`, and `foo` has it in `dependencies`.** The root's peer already guarantees that it is installed, so it stays a peer, as long as the ranges fit by the rules above.

When two shallow packages need the same dependency, the same rules apply between them: the narrower range wins, and ranges that don't overlap are a conflict.

Ranges that aren't semver, such as `workspace:`, `catalog:`, `npm:` aliases, `file:`, `link:`, git or URL ranges, are not compared. They are copied only when the root doesn't declare the dependency yet, and only `workspace:` and `catalog:` are allowed. The package manager replaces those with real versions on publish. Anything else is an error, because it would not install from the registry.

The write keeps the file's indentation and trailing newline, and inserts new keys in alphabetical order. The plugin writes only when the content changes, so watch mode doesn't rebuild in a loop. `'check'` compares the same result and fails with a diff.

### 6. Dependencies that are no longer needed

The plugin never removes entries. Once written, a hoisted dependency can't be told apart from one the author added by hand, and removing a hand-written dependency would be worse than leaving a stale one.

Instead, after the merge the plugin lists every `dependencies` entry that no output chunk imports, and logs it as a warning. This also catches unused dependencies unrelated to shallow bundling. It is a warning, not an error, because a dependency can be used outside the bundle, for example by a `bin` script that isn't built, or by consumers through a peer setup.

## Alternatives considered

- **Bundle everything** (tsdown's default for dev dependencies). This is the behavior this RFC avoids, for the reasons in [Motivation](#motivation).
- **`bundleDependencies` in `package.json`.** npm then ships `foo` with its whole `node_modules` tree in the tarball. That duplicates even more than bundling, and pnpm handles it poorly.
- **A hand-written `deps.neverBundle` function.** Rolldown passes the importer to `external`, so an author can keep `foo`'s imports external by hand. But they must also keep `package.json` in sync by hand, with no version checks. This plugin is that function, plus the parts that are easy to get wrong.
- **Publishing `foo`.** Often the right answer, but not possible for private workspace packages, and not what the author wants when the goal is to hide or pin `foo`.

## Unresolved questions

1. **A package-level hook in tsdown.** Writing in the last format's `build:done` works, but the plugin has to count formats itself, and tsdown's `publint` and `attw` checks run before the write. A hook that fires once per `package.json`, after tsdown's own `package.json` step, would be cleaner. This could be proposed upstream.
2. **Recording hoisted dependencies.** tsdown already writes an `inlinedDependencies` field listing bundled packages, `foo` included. A similar record of what was hoisted would let the plugin remove stale entries safely, at the cost of one more field in `package.json`.
3. **Default `pkg` mode.** Writing locally and checking in CI keeps the manifest in sync without extra steps. But some authors may expect a build to never touch tracked files unless they opt in, which would make `false` the safer default.
4. **Depth.** Should `include` allow `foo>bar`, meaning bundle `bar` only when `foo` imports it? It resolves conflicts in [Merging into `package.json`](#5-merging-into-packagejson) without bundling `bar` everywhere, but it adds a lot of complexity, and it could be added later without breaking changes.

## Implementation plan

```text
src/
  index.ts     # shallow() factory and options; wires the hooks together
  owner.ts     # finds shallow package roots and which package a module belongs to
  resolve.ts   # the resolveId 'pre' hook from step 3
  collect.ts   # reads external imports from output chunks (step 4)
  pkg.ts       # merges and checks package.json, compares ranges with semver (step 5)
tests/
  fixtures/
    basic/         # foo -> zod: zod ends up in dependencies
    root-range/    # the root range is narrowed, kept, or reported
    conflict/      # ranges that don't overlap
    peer/          # foo's peers become root peers
    undeclared/    # foo imports a package it never declared
    pnpm-nested/   # .pnpm store paths and nested node_modules
    workspace/     # a symlinked workspace package with workspace: and catalog: ranges
    dts/           # @types packages reached through declaration files
    multi-format/  # esm + cjs build, package.json written once
```

Runtime dependencies: `semver` for range checks. `tsdown` is a peer dependency (`>=0.21.9`, the first version with `tsdownConfig` hooks).
