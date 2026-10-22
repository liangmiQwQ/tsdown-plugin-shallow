# tsdown-plugin-shallow Agent Guide

tsdown-plugin-shallow is a tsdown plugin for library authors who want to bundle one dependency's own code, but keep that dependency's dependencies external. The external imports left in the output are merged into the library's `package.json`, so the published package still installs everything it needs.

## Project Overview

tsdown's built-in deps plugin decides what to externalize only from the root `package.json`, regardless of which module is importing. This plugin decides by the importer instead: an import made from inside a shallow-bundled package stays external when that package declares it as a dependency.

## Toolchain

Vite+ is used as the project manager and dev toolchain for JavaScript part. Check `node_modules/vite-plus/docs` if you don't know how to use Vite+ features. When you find yourself needing a dev tool a tool is missing, you can check Vite+'s document first.

## Rules

Keep JavaScript dependency versions in the default catalog in `pnpm-workspace.yaml`, and reference them with `catalog:` in package manifests.

As a opensource project, not all contributors are required to install Vite+ as `vp` globally, so when you are adding a script / task, using `package.json#scripts` to ensure the accessiblility for external contributors.

If you find AGENTS.md is outdated, please notice users to change in response.

Keep code functional. Write simple code that junior developers can understand, and make functions reusable if possible. Use Unix philosophy to design your code (Every function should only do one thing and should not be too long or complex).

Use existing dependencies and tools. Feel free to add dependencies. Don't reinvent the wheel.

Commit messages and PR titles follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

Add `.gitkeep` file when creating new empty directory.
