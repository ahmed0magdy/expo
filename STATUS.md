STATUS: DONE - #46415 runtime version cache pushed to contrib/cli-cache-runtime-version (no PR opened)
UPDATED: 2026-10-14T05:54:45Z
BRANCH: contrib/cli-cache-runtime-version (commit 958cee28, on main befa5dd8)
MODE: Auto

> **Latest update:** I've received your reminder and will follow it. I won't open, reopen or comment on any pull request or issue, on the fork or upstream, and I won't schedule check-ins, wakeups or loops. No PR or issue was opened and no check-in was scheduled during this task.
> `contrib/cli-cache-runtime-version` matches local commit `958cee28` and the working tree has no uncommitted changes. Nothing changed since the report below. The session is now idle.

# #46415: cache the fingerprint runtime version in the dev server

**Done.** One commit, `958cee28`, is on `contrib/cli-cache-runtime-version`:
- authored by `ahmed0magdy <110637215+ahmed0magdy@users.noreply.github.com>`;
- trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`, no `Claude-Session:` trailer;
- no PR opened.

## Check results (exact)

| Check | Result |
|---|---|
| New and changed tests: `npx jest src/start/server/middleware` | 22 suites, 231 tests passed |
| Full `@expo/cli` Jest suite: `npx jest`, run twice with the change | 253 suites passed; 2,223 tests passed, 7 skipped |
| `et check-packages @expo/cli` | 43/44 tasks passed. Build, typecheck and lint passed. Test task: 1 failed, 2,222 passed, 7 skipped |
| `pnpm lint --fix` (in `packages/@expo/cli`) | Exit 0. One warning, in `withMetroCacheVary.ts`, which this change doesn't touch |
| `pnpm format` | Exit 0 (it reformatted the new files before the commit) |
| Changeset | `.changeset/cli-cache-runtime-version.md`, `'@expo/cli': patch` |

**The one `et check-packages` test failure isn't caused by this change.**
- `src/start/doctor/dependencies/__tests__/validateDependenciesVersions-test.ts` fails the same way on **unmodified main** under `et check-packages` (1 failed, 2,195 passed).
- It passes under plain `npx jest`, both on main and with this change.
- That test makes a live, unmocked `fetch` to the npm registry (`fetchReactNativeTvDistTagsAsync` in `reactNativeTv.ts`) and expects `0.85-stable`. When the request fails it gets `latest` instead.
- My attempt to force the failure by breaking the proxy didn't work, because Node's built-in `fetch` ignores `HTTPS_PROXY`. So the network explanation is the likely cause, not a proven one.

**How `et` was run:** this VM has pnpm 10.28, but the repo requires pnpm 12.2.1. `et` was run with a pnpm 12.2.1 shim first on `PATH`; no global install.

**Not done:**
- **Native builds and running the app** were out of scope by your rules, so there's no simulator check of the dev server.
- **Docs:** the new env var isn't added to the env var table in `docs/pages/more/expo-cli.mdx`. CONTRIBUTING suggests docs in a separate PR. Tell me if you want it in this one.
- **Upstream PR #50396:** I couldn't read it from here, because the proxy only serves the fork. This change doesn't touch expo-updates.

---

# PR notes (raw material: rewrite in your own words)

## Design summary (for the comment on #46415)
- **Problem:** with `runtimeVersion: { policy: 'fingerprint' }`, every manifest request spawns `expo-updates runtimeversion:resolve`, which hashes the whole project again.
- **Cache:** the dev server keeps the resolved runtime version for each platform, keyed by a hash of the app config. The app config is already re-read on every manifest request, so any config change gives a new key and a fresh resolve.
- **Invalidation:** the dev server watches exactly the paths the fingerprint hashed.
  - The list comes from the `fingerprintSources` that `runtimeversion:resolve` already prints, so there is no second fingerprint run.
  - Ignore rules come from `@expo/fingerprint`'s own `normalizeOptionsAsync`: `DEFAULT_IGNORE_PATHS`, `.fingerprintignore`, `fingerprint.config.js` `ignorePaths`, and the CNG/managed `ios/**` and `android/**` ignores.
  - Ignored directories (`ios/Pods`, `ios/build`, `android/.gradle`, `android/app/build`, `.cxx`, nested `node_modules`) are never walked or watched.
- **What is watched:**
  - **`dir` sources:** every non-ignored directory, one non-recursive watch each.
  - **`file` sources:** the file's directory.
  - **`package` sources:** the package directory, and each parent up to `node_modules`, for installs and removals.
  - **Project root and Metro server (workspace) root:** non-recursive. Their files feed the hashed contents that have no path of their own: app config, `package.json`, `.env*`, `fingerprint.config.js`, `.fingerprintignore`, lockfiles.
- **No missed changes between resolving and watching:** after the watchers start, each watched directory and input file is checked with `stat`. If anything changed after the resolve began (2s slack for coarse timestamps), the result is returned but not cached.
- **Concurrency:** concurrent requests share one in-flight promise. A failed resolve is evicted, so the next request retries.
- **Fail safe:** if the watcher can't start or errors later, caching is off for the rest of the session and the server goes back to resolving on every request, with one debug event (`manifest:runtime_version_cache_disabled`). The causes covered:
  - ENOSPC or permissions;
  - `@expo/fingerprint` not resolvable through `expo`;
  - more than 2,000 directories to watch.
- **Scope:** only the `fingerprint` policy is cached. Other policies are not cached across requests; concurrent requests still share one resolve. Their result also depends on workflow detection (do native dirs exist, are they gitignored), and they are cheap.
- **Kill switch:** `EXPO_NO_RUNTIME_VERSION_CACHE=1` brings back today's behaviour.
- **Node support:**
  - `expo` supports Node `^22.13 || ^24.3 || ^26 || >=27`.
  - Recursive `fs.watch` exists on macOS and Windows natively, and on Linux on all of these versions.
  - On Linux, though, Node builds it by walking the whole tree, with no way to skip ignored directories. Under `ios/Pods` that could hit the inotify limit (ENOSPC).
  - So the change uses only non-recursive `fs.watch` over a pruned walk, which behaves the same on all three platforms.
- **Remaining gap:** a hand edit inside `node_modules`, below a package's root directory, that changes the fingerprint without changing a watched file. With the default preset, packages are hashed by `name@version`, so such an edit matters only through autolinking config, for example a podspec in a subfolder. Fix: restart the dev server, or set `EXPO_NO_RUNTIME_VERSION_CACHE=1`.

## Why
- Fixes expo/expo#46415. The dev server recomputes `@expo/fingerprint` on every manifest request when `runtimeVersion` uses the fingerprint policy, which slows down every app launch and reload against the dev server.

## How
- `resolveRuntimeVersionWithExpoUpdatesAsync.ts`: returns the full `runtimeversion:resolve` result (`runtimeVersion`, `fingerprintSources`, `workflow`) instead of only the version.
- `RuntimeVersionCache.ts` (new): per-platform entry keyed by the app config hash. Shares in-flight resolves, evicts failures, keeps fingerprint results only while their watcher is alive, and disables itself on watcher failure.
- `watchFingerprintSourcesAsync.ts` (new): builds the watch set from the sources and fingerprint's ignore rules, starts non-recursive watchers (`persistent: false`), filters events for ignored paths, and does the post-start timestamp check.
- `ExpoGoManifestHandlerMiddleware.ts`: uses the cache. The config-plugin fallback for "no expo-updates" is unchanged.
- `env.ts` adds `EXPO_NO_RUNTIME_VERSION_CACHE`; `events.ts` adds `manifest:runtime_version_cache_disabled`; plus a changeset.

## Test Plan
- **Written first and seen failing:** the cache and watcher tests failed before the modules existed. The repeated-manifest-request test failed against unchanged code.
- **`RuntimeVersionCache-test.ts`, 12 tests:**
  - repeated requests resolve once; platforms are resolved separately;
  - concurrent requests share one call; a failed resolve is evicted;
  - an app config change, and a watched source change, each trigger a fresh resolve;
  - a result whose sources changed during resolve is not cached;
  - a watcher start failure, and a later watcher error, each disable the cache for the session;
  - non-fingerprint policies only share in-flight calls; no expo-updates returns `null` uncached;
  - `EXPO_NO_RUNTIME_VERSION_CACHE=1` resolves on every request.
- **`watchFingerprintSourcesAsync-test.ts`, 14 tests, real temp project and real `@expo/fingerprint` ignore rules:**
  - the watch set excludes `ios/Pods`, `ios/build` and nested `node_modules`;
  - `.fingerprintignore` is respected; managed projects don't descend into `ios/`;
  - package and file sources watch the right directories;
  - ignored-path events are dropped; root-level config changes count;
  - a real `fs.watch` reports a real file edit;
  - "changed while resolving" returns `null`; ENOENT returns `null`;
  - ENOSPC throws and closes the open watchers; the directory limit throws;
  - a later watcher error is reported; `@expo/fingerprint` failing to load throws.
- **`ExpoGoManifestHandlerMiddleware-test.ts`:** 3 manifest requests with a fingerprint result call `runtimeversion:resolve` once.
- **Not tested:**
  - a real dev server with a device, or EAS;
  - macOS and Windows watcher behaviour (only Linux was available);
  - real `@expo/fingerprint` resolution through `expo` from an outside project (under Jest the resolver finds the monorepo copy, so that path is covered with an injected loader).

## Alternatives for maintainers
- **B, reuse existing signals only:** Metro file-watcher events plus the config key. No new watcher, but Metro drops events for `.swift`, `.kt`, `.gradle`, `.pbxproj`, `Podfile` and `patches/*.patch` (see the `hasWatchedExtension` filter in `@expo/metro-file-map/src/index.ts`), so native-only edits could serve a stale runtime version until the next JS/config change or a restart.
- **C, opt-in only:** keep today's behaviour by default, with in-flight sharing always on, and enable caching through an env var. Safest, but most users wouldn't benefit.

## AI assistance disclosure
- This change was written with AI assistance (Claude Code). I reviewed the design and the diff, and the commit carries a `Co-Authored-By: Claude` trailer.

---

## Explain it to the owner

### 1. The bug, in plain words
When your app uses `"runtimeVersion": { "policy": "fingerprint" }`, the runtime version is a hash of everything native in the project (native folders, native packages, config). Every time a device asks the dev server for the app manifest (each launch or reload), the server recomputed that whole hash from scratch.
- **Input:** open the app 3 times without changing anything.
- **Wrong behaviour:** 3 full fingerprint computations; each can take seconds on a big project.
- **Right behaviour:** 1 computation, then the same answer from memory until something it depends on actually changes.

### 2. How the fix works, step by step
1. A manifest request arrives. `ExpoGoManifestHandlerMiddleware._getManifestResponseAsync` calls `RuntimeVersionCache.getRuntimeVersionAsync(platform, exp)` instead of calling expo-updates directly.
2. `RuntimeVersionCache` hashes the app config (`exp`, re-read on every request anyway). If it has a still-valid entry for this platform with the same config hash, it returns it, even if that entry is still being computed. That's how concurrent requests share one call.
3. Otherwise it runs `resolveRuntimeVersionWithExpoUpdatesAsync`, which spawns `expo-updates runtimeversion:resolve` as before. That command already prints the list of files and folders the fingerprint used (`fingerprintSources`); we now read that list instead of throwing it away.
4. If the policy isn't `fingerprint` (no source list), the answer isn't kept. If resolving throws, the entry is removed so the next request retries.
5. For fingerprint results, `watchFingerprintSourcesAsync` watches those sources:
   - it asks `@expo/fingerprint` itself which paths are ignored, so it never walks into `ios/Pods` or build folders;
   - it watches each remaining folder (non-recursive watch per folder), plus the project root and workspace root, where app config, `package.json`, `.env` and lockfiles live;
   - it checks file timestamps once, to catch anything that changed while the hash was being computed.
6. When a watched path changes, the watcher calls `onChange`, the entry is dropped, and the next request recomputes.
7. If watching is impossible (too many folders, OS watcher limit, no `@expo/fingerprint`, a watcher error), the cache switches itself off for the session and everything works exactly like before.

**Why this approach and not the obvious ones:**
- **Plain "cache forever":** wrong after native edits.
- **A time-based expiry:** still wrong for its whole lifetime.
- **Reusing Metro's watcher:** Metro ignores `.swift`, `.kt`, `.gradle` and similar native files.
- **Recursive `fs.watch` on `ios/`:** on Linux this also watches `Pods/` and build folders and can hit the OS watcher limit.
- **Computing the source list ourselves:** that would mean a second fingerprint run. expo-updates already gives us the list for free.

### 3. Why it's correct, and what the tests don't prove
**The tests prove:**
- repeated, concurrent and failed requests behave as described;
- config changes and source changes both force a new resolve;
- watcher failures switch the cache off;
- the kill switch works;
- the watch set really excludes Pods, build and nested `node_modules`, using the real fingerprint ignore rules on a real temp project;
- a real file edit fires the real watcher (Linux).

**Not proven by tests:**
- watcher behaviour on macOS and Windows;
- an end-to-end run with a real dev server, device and expo-updates;
- `@expo/fingerprint` being found through `expo` in a real app (it's simulated in tests);
- the remaining gap: hand edits deep inside `node_modules`.

### 4. Questions a maintainer will likely ask
1. **"Why not just use Metro's file watcher?"** Metro only reports JS/asset file types. Native files like `.swift` and `.gradle` never produce events there (`hasWatchedExtension` in `@expo/metro-file-map`).
2. **"Can this exhaust inotify watches like Metro sometimes does?"** It watches only non-ignored folders of the fingerprint sources, never `Pods` or build output. It stops at 2,000 folders and then disables itself instead of failing.
3. **"What if a file changes while the fingerprint is being computed?"** After the watchers start, timestamps are compared with the moment resolving began. Anything newer means the result is returned but not cached.
4. **"Why aren't other runtime version policies cached?"** Their answer also depends on workflow detection (do `ios/` and `android/` exist, are they gitignored), which isn't part of the config key. They're cheap, so they stay uncached; concurrent requests still share one call.
5. **"How do I turn it off, or what if it's wrong?"** Set `EXPO_NO_RUNTIME_VERSION_CACHE=1`, or restart the dev server. The only known gap is a hand edit deep inside `node_modules`.

### 5. Risks, and how each was checked
- **Stale runtime version:** the main risk. Guarded by the config key, the watcher on every hashed path, and the timestamp check. Tested for config changes, source changes and changes during resolve.
- **Too many OS watchers:** pruned with fingerprint's own ignores, capped at 2,000, and fails safe. Tested with a cap and with a simulated ENOSPC.
- **Watchers keeping the process alive:** all watchers use `persistent: false`.
- **Windows:** deleting a watched folder can make the watcher error, which only disables the cache for the session. Not run on Windows.
- **Breaking the existing manifest flow:** the full middleware suite (22 suites, 231 tests) and the full CLI suite (2,223 tests) pass. The fallback for "no expo-updates" is unchanged.
- **The ignore rules come from the project's `@expo/fingerprint`** (through `expo`, the same copy expo-updates uses), so they match what was hashed. If it can't be loaded, the cache is off.

### 6. Read the diff in this order (about 10 minutes)
1. `resolveRuntimeVersionWithExpoUpdatesAsync.ts`: now returns the full result. Check that the defaults (`fingerprintSources ?? null`, `workflow ?? 'generic'`) make sense.
2. `RuntimeVersionCache.ts`: `getRuntimeVersionAsync` (key and sharing), `resolveAndWatchAsync` (what gets kept), `evict` and `disable`.
3. `watchFingerprintSourcesAsync.ts`: the class comment, then what is watched for each source type, the event filter, and the timestamp check at the end.
4. `ExpoGoManifestHandlerMiddleware.ts`: a one-line swap to the cache.
5. `env.ts`, `events.ts` and the changeset: the kill switch, the debug event, and the release note.
6. Tests: `RuntimeVersionCache-test.ts`, then `watchFingerprintSourcesAsync-test.ts`, then the new case at the end of `ExpoGoManifestHandlerMiddleware-test.ts`.
