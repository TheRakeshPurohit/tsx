# Module customization hooks

How tsx registers with Node's module system, and why it picks the sync or async hook API per version.

## `module.register()` — async hooks (loader worker thread)

- **What**: registers ESM `resolve`/`load` hooks that run on a dedicated **loader worker thread**. The main thread communicates with it over a `MessagePort`; synchronous hook callers block in `makeSyncRequest` ([v24.10.0 `hooks.js#L598`](https://github.com/nodejs/node/blob/v24.10.0/lib/internal/modules/esm/hooks.js#L598)) and wait on the worker notification with `AtomicsWait` ([`hooks.js#L609`](https://github.com/nodejs/node/blob/v24.10.0/lib/internal/modules/esm/hooks.js#L609)).
- **PR**: [#46826](https://github.com/nodejs/node/pull/46826)
- **Verified**: v18.19.0, v20.6.0, v21.0.0 (matches gate)
- **tsx**: `moduleRegister` gate in `src/utils/node-features.ts`; async path in `src/esm/api/register.ts` (`module.register('./esm/index.mjs?…', { data: { port } })`).
- **Cost**: the worker round-trip is the dominant fixed startup tax on this path. A profile of a large module graph using the async path spent **~36% of startup in `makeSyncRequest`** — the main thread blocked on atomics waiting for the loader worker. This is the tax the sync API removes.
- **Measured** (benchmark suite, `hooks-passthrough`, 1000 modules, M5 Pro, isolating registration + pass-through with zero transforms): same tsx build on Node 24.10.0 (async worker) vs 24.15.0 (sync hooks) = **145ms sync → 200ms async, +55ms** for the worker path. This is the pure mechanism cost of the async hook transport.

## `module.registerHooks()` — sync hooks (in-thread)

- **What**: registers `resolve`/`load` hooks that run **synchronously in the same thread** — no worker, no atomics round-trip. The public sync-hook registration entry is `registerHooks` ([v22.15.0 `customization_hooks.js#L112`](https://github.com/nodejs/node/blob/v22.15.0/lib/internal/modules/customization_hooks.js#L112), [v24.15.0 `customization_hooks.js#L111`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/customization_hooks.js#L111)). Node then routes module work through `loadWithHooks` ([v24.15.0 `customization_hooks.js#L365`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/customization_hooks.js#L365)) and `resolveWithHooks` ([`customization_hooks.js#L408`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/customization_hooks.js#L408)).
- **PR**: [#55698](https://github.com/nodejs/node/pull/55698)
- **Verified**: v22.15.0, v23.5.0, v24.0.0
- **tsx**: sync path in `src/esm/api/register.ts` (`createResolveSync`/`createLoadSync` from `src/esm/hook/resolve.ts` + `load.ts`).

## CJS-reload safety — the boundary tsx actually gates on

Sync hooks alone are not sufficient for tsx's CommonJS interop. tsx needs Node to be able to **re-enter `Module._load` from a sync ESM hook** (so a CJS module transformed by the load hook is re-run through the CJS loader, preserving `require.cache`, globals, and CJS semantics). That capability landed later than the sync-hooks API.

- **What**: sync load/translate path handles re-entry into the CJS loader (null-source handling from sync hooks). Before the boundary, v24.11.0's sync load result has no reload flag ([`load.js#L133-L158`](https://github.com/nodejs/node/blob/v24.11.0/lib/internal/modules/esm/load.js#L133-L158)), and the CJS translator does not choose the CJS loader from a sync-hook reload signal ([`translators.js#L333-L351`](https://github.com/nodejs/node/blob/v24.11.0/lib/internal/modules/esm/translators.js#L333-L351)). At v24.11.1 the load result gains `shouldBeReloadedByCJSLoader` ([`load.js#L144-L171`](https://github.com/nodejs/node/blob/v24.11.1/lib/internal/modules/esm/load.js#L144-L171)), and the translator re-enters `Module._load` when that flag or null source requires it ([`translators.js#L334-L363`](https://github.com/nodejs/node/blob/v24.11.1/lib/internal/modules/esm/translators.js#L334-L363)). The same shape is backported to v22.22.3 ([`load.js#L148-L175`](https://github.com/nodejs/node/blob/v22.22.3/lib/internal/modules/esm/load.js#L148-L175), [`translators.js#L331-L362`](https://github.com/nodejs/node/blob/v22.22.3/lib/internal/modules/esm/translators.js#L331-L362)).
- **PR**: [#59929](https://github.com/nodejs/node/pull/59929) — "module: handle null source from async loader hooks"
- **Verified**: **v22.22.3**, v24.11.1, v25.1.0
- **tsx**: `moduleRegisterHooksCjsReload` gate; `supportsRegisterHooks` in `src/esm/api/register.ts` requires this gate (not just `registerHooks` presence).
- **Related tsx machinery**: `src/utils/cjs-loader-state.ts` (`isGlobalCjsLoaderActive`) gates the sync hooks' CJS pass-through; `src/utils/ipc/client.ts` buffers early dependency messages so watch mode sees sync-hook loads before the socket connects.

### Finding: the gate omits the v22.x line

The gate lists `[24,11,1] / [25,1,0] / [26,0,0]` but **not a 22.x entry**, so `isFeatureSupported` returns false on the entire v22 line — tsx uses the async `module.register()` path there.

But #59929 **was backported to v22.22.3** (released 2026-05-11), and `registerHooks` has been in the 22 line since v22.15.0. So on **v22.22.3+**, both prerequisites are present, yet tsx still takes the slower async path.

Timeline suggests why: the gate was written in tsx `1d7e528` on **2026-05-13**, two days after v22.22.3 shipped the backport. The 22.x backport (2026-05) landed ~6 months after the original 24.x/25.x releases (2025-10/11), so it wasn't available when the sync path was first designed.

- **Candidate follow-up**: add a `[22, 22, 3]` boundary to `moduleRegisterHooksCjsReload` so v22.22.3+ users get the sync fast path. Must first confirm the 22.x backport is behaviorally complete (run the CJS-interop + watch test suites on v22.22.3), since a security/patch backport can be narrower than the original.
- Note: the gate's `[26,0,0]` entry is redundant for correctness (`[25,1,0]` already covers all of 26+ via the major-range logic in `isFeatureSupported`); it's documentation of intent, harmless.

## Worker preload caveat

`src/esm/index.ts` only auto-registers on the main thread / non-internal worker threads: on the async path via `workerThreads.isMainThread`, and on the sync path it skips Node's **internal** loader thread (`workerThreads.isInternalThread`, [#56469](https://github.com/nodejs/node/pull/56469)) where an async `module.register()` preload could otherwise evaluate tsx itself. `isInternalThread` isn't exported before v22.14, hence the namespace import that avoids a load-time `SyntaxError`.

## Decisions

### Sync `registerHooks()` only on CJS-reload-safe versions

tsx gates the sync path on `moduleRegisterHooksCjsReload` (#59929), **not** merely on `typeof module.registerHooks === 'function'` (#55698). Between v22.15.0 and the reload-safety boundary, sync hooks exist but do not yet provide the CJS reload behavior tsx needs for transformed CommonJS modules. Correctness gates the fast path; the async `module.register()` path stays the fallback for older supported versions. (Rationale: tsx `1d7e528`.)

### Stack inspection over AsyncLocalStorage for require-context detection

For detecting "this load hook is firing for a CJS `require()`" under composed loader chains, tsx uses call-stack inspection rather than wrapping `Module.prototype.require` with AsyncLocalStorage.

Measured tradeoff: stack inspection is ~200x more expensive **per call** (~3–5µs vs ~20ns) but fires rarely (only bridge JSON loads through composed hooks), whereas ALS pays a ~130ns wrapping cost on **every** `require()`. Across a realistic workload (100k requires, ~100 bridge-JSON loads) stack inspection totals ~0.5ms vs ALS's ~14ms — **~36x cheaper in aggregate**. ALS remains a good fit if tsx later needs broader tracing or telemetry; until then, the narrow stack check has the lower aggregate cost. Because it depends on Node's internal frame name, it should stay covered by a regression test.
