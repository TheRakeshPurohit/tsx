# Gate reference

Smaller Node gates that tsx forwards or mirrors, but that do not need a full mechanism story.

| Gate | Section |
| --- | --- |
| `importAttributes` | [Import attributes](#import-attributes-importattributes) |
| `importMetaPathProperties` | [`import.meta` path properties](#importmeta-path-properties-importmetapathproperties) |
| `wasmModules` | [WASM modules](#wasm-modules-wasmmodules) |
| `modulePackageMainResolution` | [`legacyMainResolve` assertion fix](#legacymainresolve-assertion-fix-modulepackagemainresolution) |
| `cliTestFlag` | [`--test` flag](#--test-flag-clitestflag) |
| `testRunnerGlob` | [Test runner glob](#test-runner-glob-testrunnerglob) |

## Import attributes (`importAttributes`)

- **What**: `import … with { type: 'json' }` (import attributes), succeeding the earlier `assert { … }` assertions syntax. The load path aliases legacy assertions into attributes at [v20.10.0 `load.js#L116-L124`](https://github.com/nodejs/node/blob/v20.10.0/lib/internal/modules/esm/load.js#L116-L124), and validation checks the `type` attribute in [`assert.js#L57-L80`](https://github.com/nodejs/node/blob/v20.10.0/lib/internal/modules/esm/assert.js#L57-L80).
- **Verified**: v18.19.0, v20.10.0, v21.0.0.
- **tsx**: `src/esm/hook/load.ts` picks the `importAttributes` vs legacy `importAssertions` context property accordingly.

## `import.meta` path properties (`importMetaPathProperties`)

- **What**: `import.meta.dirname` and `import.meta.filename`. Node initializes them from the file URL path at [v20.11.0 `initialize_import_meta.js#L61-L62`](https://github.com/nodejs/node/blob/v20.11.0/lib/internal/modules/esm/initialize_import_meta.js#L61-L62) and [v21.2.0 `initialize_import_meta.js#L61-L62`](https://github.com/nodejs/node/blob/v21.2.0/lib/internal/modules/esm/initialize_import_meta.js#L61-L62).
- **PR**: [nodejs/node#48740](https://github.com/nodejs/node/pull/48740)
- **Verified**: v20.11.0, v21.2.0.
- **tsx**: `src/utils/transform/index.ts` injects `dirname`/`filename` only where Node exposes them, avoiding a transformed-CJS shape that diverges from Node.

## WASM modules (`wasmModules`)

- **What**: `import` of `.wasm` modules. The import-attributes validator treats `wasm` as an implicit type at [v22.19.0 `assert.js#L32`](https://github.com/nodejs/node/blob/v22.19.0/lib/internal/modules/esm/assert.js#L32) and [v24.5.0 `assert.js#L32`](https://github.com/nodejs/node/blob/v24.5.0/lib/internal/modules/esm/assert.js#L32).
- **PR**: [nodejs/node#57038](https://github.com/nodejs/node/pull/57038)
- **Verified**: v22.19.0, v24.5.0.
- **tsx**: `wasmModules` gate.

## `legacyMainResolve` assertion fix (`modulePackageMainResolution`)

- **What**: an assertion in package `main` resolution (`legacyMainResolve`) threw on certain inputs; fixed for the 18.x line. The resolver entry is [v18.20.4 `resolve.js#L176`](https://github.com/nodejs/node/blob/v18.20.4/lib/internal/modules/esm/resolve.js#L176), called from package resolution at [`resolve.js#L903`](https://github.com/nodejs/node/blob/v18.20.4/lib/internal/modules/esm/resolve.js#L903). v18.20.5 keeps the same public shape ([`resolve.js#L176`](https://github.com/nodejs/node/blob/v18.20.5/lib/internal/modules/esm/resolve.js#L176), [`resolve.js#L903`](https://github.com/nodejs/node/blob/v18.20.5/lib/internal/modules/esm/resolve.js#L903)) but fixes the internal assertion; newer lines never had the broken assertion.
- **PR**: [nodejs/node#55708](https://github.com/nodejs/node/pull/55708)
- **Verified**: v18.20.5 only; 19+ treated as supported because they never carried the bug.
- **tsx**: `modulePackageMainResolution` gate.

## `--test` flag (`cliTestFlag`)

- **What**: built-in test runner CLI flag.
- **Reference**: [v18.1.0 CLI docs](https://github.com/nodejs/node/blob/v18.1.0/doc/api/cli.md).
- **Verified**: v18.1.0.
- **tsx**: forwards `--test` only where supported.

## Test runner glob (`testRunnerGlob`)

- **What**: built-in test runner accepts glob patterns for test selection.
- **Reference**: [v21.0.0 release](https://github.com/nodejs/node/releases/tag/v21.0.0).
- **Verified**: v21.0.0.
- **tsx**: forwards glob-style test args only where supported.
