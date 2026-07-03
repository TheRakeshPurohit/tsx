# CJS/ESM interop

How Node bridges CommonJS and ES modules, and how tsx shapes transformed TypeScript so it still fits Node's CJS/ESM contracts.

## Shared foundation: CJS preparse + synthetic namespaces

When ESM imports CJS, Node synthesizes an ESM namespace from static CJS source analysis. It does **not** discover named exports by running the CJS module first.

- In v24.15.0, the CJS translator calls `cjsPreparseModuleExports` before building the wrapper ([`translators.js#L212`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L212)); the function is defined at [`translators.js#L381`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L381).
- The parser returns `[exports, reexports]` ([`translators.js#L393`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L393)). Node follows reexports such as `module.exports = { ...require('x') }` recursively ([`translators.js#L398-L418`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L398-L418)).
- Node marks CJS modules cached by the ESM loader with `kIsCachedByESMLoader` ([`translators.js#L368`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L368)); the CJS loader uses that marker during circular loads ([`loader.js#L1297-L1308`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/cjs/loader.js#L1297-L1308)).
- Node v23+ adds a synthetic `'module.exports'` key to the namespace wrapper ([v23.0.0 `translators.js#L187`](https://github.com/nodejs/node/blob/v23.0.0/lib/internal/modules/esm/translators.js#L187)); newer versions preserve it unless the lexer already found that name ([v24.15.0 `translators.js#L219-L220`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L219-L220)).

The parser implementation changed but the grammar contract stayed the same:

- v24.11.1 uses vendored JS `cjs-module-lexer` ([`translators.js#L73-L82`](https://github.com/nodejs/node/blob/v24.11.1/lib/internal/modules/esm/translators.js#L73-L82)). Its README documents supported patterns (`exports.foo`, `module.exports = { a }`, reexports) at [`deps/cjs-module-lexer/README.md#L7-L51`](https://github.com/nodejs/node/blob/v24.11.1/deps/cjs-module-lexer/README.md#L7-L51) and scope-insensitive matching at [`README.md#L147-L152`](https://github.com/nodejs/node/blob/v24.11.1/deps/cjs-module-lexer/README.md#L147-L152).
- v24.14.0 switches to native `internalBinding('cjs_lexer')` via [nodejs/node#61456](https://github.com/nodejs/node/pull/61456) ([`translators.js#L72`](https://github.com/nodejs/node/blob/v24.14.0/lib/internal/modules/esm/translators.js#L72)). Merve documents the same export/reexport grammar at [`merve.h#L82-L102`](https://github.com/nodejs/node/blob/v24.14.0/deps/merve/merve.h#L82-L102), parses exports/reexports in [`merve.cpp#L630-L676`](https://github.com/nodejs/node/blob/v24.14.0/deps/merve/merve.cpp#L630-L676), and returns the `[Set, Array]` shape from [`src/node_cjs_lexer.cc#L44-L87`](https://github.com/nodejs/node/blob/v24.14.0/src/node_cjs_lexer.cc#L44-L87).

tsx depends on the grammar, not the implementation. Grammar changes are what to re-check when verifying named-export interop on a new Node line.

## ESM importing CJS

Two gates control this direction:

| Gate | Node change | Node PR(s) | Verified releases | tsx use |
| --- | --- | --- | --- | --- |
| `esmLoadReadFile` | The ESM `load` hook can return source for `format === 'commonjs'`. | [nodejs/node#50825](https://github.com/nodejs/node/pull/50825) | v20.11.0, v21.3.0 | `src/esm/hook/load.ts` can transform CJS before Node evaluates/preparses it. |
| `cjsNamespaceFromLoadHook` | CJS source returned by the load hook can be preparsed into a namespace. | [nodejs/node#50825](https://github.com/nodejs/node/pull/50825), [#54769](https://github.com/nodejs/node/pull/54769) | `[20.11.0, 21.0.0)` and `>=21.3.0` | tsx can preserve named exports from transformed CommonJS TypeScript. |

Node's load hook passes import attributes context and reads CJS source at the boundary ([v20.11.0 `load.js#L113-L124`](https://github.com/nodejs/node/blob/v20.11.0/lib/internal/modules/esm/load.js#L113-L124), [`load.js#L145`](https://github.com/nodejs/node/blob/v20.11.0/lib/internal/modules/esm/load.js#L145); [v21.3.0 `load.js#L145`](https://github.com/nodejs/node/blob/v21.3.0/lib/internal/modules/esm/load.js#L145)). v20.11.0's translator then preparses CJS before namespace creation ([`translators.js#L190`](https://github.com/nodejs/node/blob/v20.11.0/lib/internal/modules/esm/translators.js#L190)) and evaluates via `CJSModule._load` ([`translators.js#L203`](https://github.com/nodejs/node/blob/v20.11.0/lib/internal/modules/esm/translators.js#L203)).

tsx uses this by transforming TypeScript to JavaScript before Node preparses it. The important emitted shape is esbuild's dead-code CJS export annotation:

```text
0 && (module.exports = { namedExport });
```

The annotation never runs, but Node's CJS lexer recognizes it. `parentImportsCommonJsExports` detects when a parent imports named or namespace exports from a CJS target; `resolve.ts` adds `tsx-commonjs-export-preparse=1`; `load.ts` sees that query and returns transformed JavaScript so Node's preparse step can build the right namespace. The query is stripped from user-visible URLs after it has coordinated resolve/load/cache behavior.

## CJS requiring ESM

Node's `require(esm)` support is a set of overlapping windows:

| Gate | Node behavior | Node PR(s) | Verified releases / window | tsx use |
| --- | --- | --- | --- | --- |
| `requireEsm` | CJS `require()` can load eligible ESM instead of throwing `ERR_REQUIRE_ESM`. Before: `.mjs` throws ([v20.18.0 `loader.js#L1285`](https://github.com/nodejs/node/blob/v20.18.0/lib/internal/modules/cjs/loader.js#L1285)); after: `loadESMFromCJS` handles it ([v20.19.0 `loader.js#L1310`](https://github.com/nodejs/node/blob/v20.19.0/lib/internal/modules/cjs/loader.js#L1310), [`loader.js#L1509-L1511`](https://github.com/nodejs/node/blob/v20.19.0/lib/internal/modules/cjs/loader.js#L1509-L1511)). | [nodejs/node#55085](https://github.com/nodejs/node/pull/55085) | v20.19.0, v22.12.0, v23.0.0 | Enables native-style `require(esm)` interop in transformed TS. |
| `requireEsmNoWarning` | Normal `require(esm)` stops printing the experimental warning. v22.12 emits it ([`loader.js#L1404`](https://github.com/nodejs/node/blob/v22.12.0/lib/internal/modules/cjs/loader.js#L1404)); v22.13 keeps it behind tracing ([`loader.js#L1401-L1405`](https://github.com/nodejs/node/blob/v22.13.0/lib/internal/modules/cjs/loader.js#L1401-L1405)). | [nodejs/node#56194](https://github.com/nodejs/node/pull/56194) | v20.19.0, v22.13.0, v23.5.0 | Version-sensitive warning expectations. |
| `requireEsmExtensionlessMjs` | Bug window for extensionless specifiers resolving to `.mjs`. Broken window filters `.mjs` from extensionless lookup ([v20.19.0 `loader.js#L440`](https://github.com/nodejs/node/blob/v20.19.0/lib/internal/modules/cjs/loader.js#L440), [`loader.js#L654-L658`](https://github.com/nodejs/node/blob/v20.19.0/lib/internal/modules/cjs/loader.js#L654-L658); [v22.12.0 `loader.js#L676-L680`](https://github.com/nodejs/node/blob/v22.12.0/lib/internal/modules/cjs/loader.js#L676-L680)). Fixed by passing resolved format/source to `loadESMFromCJS` ([v20.19.5 `loader.js#L1303`](https://github.com/nodejs/node/blob/v20.19.5/lib/internal/modules/cjs/loader.js#L1303), [`loader.js#L1503`](https://github.com/nodejs/node/blob/v20.19.5/lib/internal/modules/cjs/loader.js#L1503); [v22.14.0 `loader.js#L1325`](https://github.com/nodejs/node/blob/v22.14.0/lib/internal/modules/cjs/loader.js#L1325), [`loader.js#L1536`](https://github.com/nodejs/node/blob/v22.14.0/lib/internal/modules/cjs/loader.js#L1536)). | [nodejs/node#55085](https://github.com/nodejs/node/pull/55085), [#55590](https://github.com/nodejs/node/pull/55590) | `[20.19.0, 20.19.5)`, `[22.12.0, 22.14.0)` | `isFeatureSupportedInRange` gate for the broken window only. |
| `cjsNamespaceModuleExports` | Synthetic CJS namespaces expose `'module.exports'`. | [nodejs/node#57366](https://github.com/nodejs/node/pull/57366) later fixes this area | v23.0.0 feature | Supports tsx's `module.exports` unwrap semantics. |

For transformed ESM required from CJS, tsx mirrors Node's `export { value as "module.exports" }` escape hatch. esbuild emits ESM exports as accessor descriptors; ordinary CJS object-literal exports are data descriptors. `src/cjs/api/module-extensions.ts` reads `Object.getOwnPropertyDescriptor(exports, 'module.exports')` and unwraps only when the descriptor is a getter and the file is a native `require(esm)` candidate. This keeps ordinary CJS objects with a literal `"module.exports"` key intact.

## Decisions

tsx lets Node own namespace construction wherever possible. It transforms TypeScript only far enough for Node's own lexer/translator to see JavaScript that matches Node's documented internal grammar. That is why the implementation prefers transform-before-preparse and descriptor-based unwrapping over a parallel namespace construction algorithm.

## Implementation history in tsx

- [7c85303](https://github.com/privatenumber/tsx/commit/7c85303) introduced named import from CJS support: the key issue was Node namespace construction, not CJS execution.
- [807f467](https://github.com/privatenumber/tsx/commit/807f467) showed why decorator/TypeScript syntax must be transformed before the lexer sees it.
- [11de737](https://github.com/privatenumber/tsx/commit/11de737) kept export annotation on the original file URL and returned transformed source so Node can preparse named exports while preserving relative resolution and `import.meta.url`; it also added the parent-import-shape detection that became `parentImportsCommonJsExports`.
- [cf8f199](https://github.com/privatenumber/tsx/commit/cf8f199) added descriptor-based `module.exports` unwrapping for native `require(esm)` parity while avoiding false positives for ordinary CJS object keys.
