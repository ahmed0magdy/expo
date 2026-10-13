STATUS: WORKING - #46415: design done, writing failing tests next
UPDATED: 2026-10-13T07:29:33Z
BRANCH: contrib/cli-cache-runtime-version (local only, no commits yet)
MODE: Auto

## Status

Building option A for expo/expo#46415. Research is done; next is the failing tests, then the implementation.

Findings that shape the design:
- **The source list comes free.** `expo-updates runtimeversion:resolve` already prints `fingerprintSources` and `workflow` alongside `runtimeVersion`. The dev server can therefore take its watch set from the exact sources the fingerprint just hashed, with no second fingerprint run.
- **Ignores come from fingerprint itself.** They come from `@expo/fingerprint`'s `normalizeOptionsAsync`: `DEFAULT_IGNORE_PATHS`, `.fingerprintignore`, `fingerprint.config.js` `ignorePaths`, and the CNG/managed `ios/**` and `android/**` ignores. Pods/, build/, .gradle/ and .cxx/ are therefore never watched.
- **Supported Node versions:** `^22.13 || ^24.3 || ^26 || >=27` (from the `expo` package `engines`).
  - Recursive `fs.watch` exists on macOS and Windows natively, and on Linux on all of these versions.
  - On Linux, though, Node walks the whole tree with no way to skip ignored directories.
  - So the design uses one non-recursive watch per directory over a pruned walk, which works the same on every platform.
- **`@expo/fingerprint` is only a devDependency of `@expo/cli`.** It is resolved from the project through `expo`, the same copy expo-updates uses. If it can't be resolved, the cache is disabled for the session.
- **Upstream PR #50396 couldn't be read here** (the proxy only serves the fork). This change doesn't touch expo-updates.

Next update when the tests and implementation are pushed.
