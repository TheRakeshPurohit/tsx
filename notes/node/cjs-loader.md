# CommonJS loader internals

How Node's CommonJS resolver probes files, how ESM error decoration re-enters it, and why tsx tries to avoid exception-driven candidate probing on hot paths.

## `Module._resolveFilename` / `Module._findPath`

- **What**: `Module._resolveFilename(request, parent, ...)` is the CommonJS resolution entry point. In v24.15.0 it is defined in [`lib/internal/modules/cjs/loader.js#L1392`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L1392).
- **Path search**: `_resolveFilename` delegates to `Module._findPath`, defined at [`loader.js#L704`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L704). `_findPath` builds a cache key, checks `Module._pathCache` ([initialization](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L353), [lookup](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L713), [store](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L803)), then probes candidate files and directories.
- **Implicit extension probing**: `tryExtensions(basePath, exts, isMain)` is [`loader.js#L575`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L575). This is where a request like `./target.js` can become `./target.js.ts` if `.ts` is registered in `Module._extensions`; that is Node's native CommonJS extension probing, not ESM resolution.
- **tsx**: `src/cjs/api/module-resolve-filename/index.ts` patches `_resolveFilename` through `createResolveFilename` to prioritize TypeScript candidates and tsconfig path aliases. Candidate generation lives in `src/utils/map-ts-extensions.ts`; candidate probing in `src/cjs/api/module-resolve-filename/resolve-ts-extensions.ts`; implicit `index` behavior in `resolve-implicit-extensions.ts`. The ESM API also installs a thin `_resolveFilename` wrapper in `src/esm/api/register.ts` so data-URL CJS interop can map back to the original path.

## `Module._cache` and data-URL path restoration

Node's CommonJS cache is keyed by the resolved filename. `_load` checks `Module._cache[filename]` before creating or evaluating a module ([v24.15.0 `loader.js#L1250-L1256`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L1250-L1256)), creates a new `Module(filename, parent)` when absent ([`loader.js#L1323`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L1323)), and treats modules cached by the ESM loader specially during circular loads ([`loader.js#L1297-L1308`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L1297-L1308)).

tsx uses three cache-coordination tricks around that contract:

- `src/esm/hook/load.ts` sometimes returns a `data:text/javascript,...?filePath=<original>` response URL for transformed CJS. Node compiles the CJS wrapper with the URL it receives ([v24.15.0 `translators.js#L112-L117`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L112-L117)), so tsx encodes the original file path in the data URL and later restores it.
- `src/cjs/api/module-resolve-filename/interop-cjs-exports.ts` maps that data URL back to the original file path and moves the `Module._cache` entry.
- `src/cjs/api/module-resolve-filename/preserve-query.ts` and `src/cjs/api/module-extensions.ts` move cache entries between query-bearing and clean paths so one physical module keeps one CJS identity while tsx still passes internal query state between hooks.

This is why query parameters used by the loader are stripped from user-visible URLs but preserved through the private cache path long enough for CJS export parsing and reload semantics.

## `Module._extensions` / `require.extensions`

Node initializes the extension handler table at [`loader.js#L355`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L355). When loading a file, it chooses the longest registered extension in the filename ([`findLongestRegisteredExtension`, `loader.js#L591-L600`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L591-L600)) and dispatches to `Module._extensions[extension]` ([`loader.js#L1545-L1553`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L1545-L1553)). The default `.js` loader is registered at [`loader.js#L1926`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L1926).

tsx patches this table in `src/cjs/api/module-extensions.ts`:

- explicit TypeScript extensions (`.ts`, `.tsx`, `.cts`, `.mts`, `.jsx`) are routed through the transformer;
- explicit unknown extension cases can fall through the `.js` handler path when Node dispatches by longest registered extension;
- tsx keeps `.ts`/`.tsx`/`.jsx` registrations enumerable because tools such as rechoir/webpack-cli inspect `require.extensions` by enumeration, not only lookup.

This explains a subtle behavior: in CommonJS, `require('./target.js')` can resolve `target.js.ts` if `.ts` is registered, because Node's native `tryExtensions` appends registered extensions to the requested base ([`tryExtensions`, `loader.js#L575`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L575)). That is distinct from tsx's ESM TypeScript extension mapping.

## Eager CommonJS-hint decoration on ESM resolution errors

- **What**: when ESM resolution fails with `ERR_MODULE_NOT_FOUND` or `ERR_UNSUPPORTED_DIR_IMPORT`, Node eagerly tries to produce a CommonJS suggestion. In v24.15.0, `defaultResolve` catches the error and calls `decorateErrorWithCommonJSHints` at [`esm/resolve.js#L994-L1004`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/resolve.js#L994-L1004).
- **Cost**: `decorateErrorWithCommonJSHints` is [`esm/resolve.js#L1022`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/resolve.js#L1022). It calls `resolveAsCommonJS`, defined at [`esm/resolve.js#L870`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/resolve.js#L870), which constructs a temporary CJS module and runs `CJSModule._resolveFilename`.
- **Consequence**: a loader that probes candidates by calling `nextResolve(candidate)` and catching misses pays **two** resolution paths for each miss: the failed ESM resolution and the eager CJS hint resolution. If the process has patched `_resolveFilename`, the hint path re-enters that patch too.
- **tsx**: tsx's resolver minimizes both the number and the cost of misses. Verbatim TypeScript extensions (`.ts`/`.tsx`/`.mts`/`.cts`) are not expanded into appended guesses, swap-list extensions do not get appended guesses such as `x.js.ts`, and file candidates that provably do not exist are skipped before calling `nextResolve` or `_findPath`.

## CJS namespace preparsing

The CJS preparse and synthetic namespace pipeline is covered in [cjs-esm-interop.md](./cjs-esm-interop.md#shared-foundation-cjs-preparse--synthetic-namespaces). This file only tracks the CJS loader mechanics that feed that pipeline: path resolution, extension handlers, cache identity, and eager ESM error decoration.

## Candidate upstream improvement: lazy CJS-hint decoration

Because Node computes the CommonJS hint at throw time, every caught-and-discarded `ERR_MODULE_NOT_FOUND` pays a full CJS resolution even though no user ever sees the hint. A possible upstream performance improvement is to compute that hint lazily, only when the error's message or stack is observed, or only at the top-level resolution boundary where the error is known to surface.

This is a Node mechanism issue, not tsx-specific. tsx works around it by reducing avoidable misses, but any loader that probes alternative specifiers can hit the same eager-decoration cost.

## Decisions

### Candidate hygiene over exception-driven probing

tsx resolves TypeScript extensions by trying candidates and catching expected misses. The resolver keeps that shape because Node remains the source of truth for real resolution, package exports, and symlinks, but it avoids candidates that cannot win:

- specifiers already ending in a verbatim TS extension resolve as-is;
- `.js`/`.jsx`/`.cjs`/`.mjs` swap candidates do not get extra appended-extension guesses;
- cheap existence checks skip missing file candidates before invoking Node's expensive failed-resolution path.

Rejected alternative: detecting the `resolveAsCommonJS` call frame and skipping work only during error decoration. That would depend on an internal frame name and only treat one symptom. Candidate hygiene reduces the miss count and cost everywhere. The behavior is protected by resolution-priority tests in `tests/specs/resolution-priority.ts`.

## Implementation history in tsx

- [e1464cf](https://github.com/privatenumber/tsx/commit/e1464cf) (`feat(cjs): support query for cache busting`) introduced CJS query handling. That is the root of the cache-key coordination problem: Node caches by filename, but tsx sometimes needs query-bearing specifiers to represent distinct transformed module identities.
- [1603c66](https://github.com/privatenumber/tsx/commit/1603c66) (`refactor(cjs): cjs lexer detection`) introduced the dedicated detection around Node's CJS lexer/preparse path. This explains why `is-from-cjs-lexer.ts` exists instead of treating all CJS resolution calls the same: resolution triggered by Node's preparse path needs query/cache handling that ordinary user `require()` does not.
- [0329bfc](https://github.com/privatenumber/tsx/commit/0329bfc) (`fix(cjs): patch module.path for accurate cache ID`) shows another private CJS-loader dependency: tsx must keep `module.path` and cache identity aligned with the cleaned filename, or CJS modules with query/cache-busting identities resolve relatives incorrectly.
- [dba2207](https://github.com/privatenumber/tsx/commit/dba2207) (`fix: decode require(esm)-bridge-encoded URLs in ESM resolve hook`) documents a Node 24 sync-bridge edge: Node converted a namespace-bearing filename to a file URL, percent-encoding `?` into the pathname (`%3F`). tsx now decodes that bridge artifact before default ESM resolution so Node resolves the underlying file while tsx keeps namespace inheritance.
- [1ce8463](https://github.com/privatenumber/tsx/commit/1ce8463) (`fix: resolve CommonJS directory requires inside dependencies`) documents why CJS `require()` contexts sometimes must be deferred to Node unchanged. A dependency's directory-style CJS request (`require('..')`, `require('process/')`) flowing through sync hooks needs to stay in Node's CJS resolver long enough for package/directory semantics to apply.

This history is why the CJS resolver code has several narrow-looking branches: they are preserving distinct Node identities (cache filename, query-bearing loader identity, CJS preparse identity, and package/directory CJS semantics) that Node itself keeps separate internally.
