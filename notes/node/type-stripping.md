# Native TypeScript type stripping

Node's built-in TypeScript execution pipeline, how it evolved, and which remaining TypeScript runtime gaps tools such as tsx still fill.

## Timeline

| Change | PR / issue | Verified releases | Why it matters to tsx |
| --- | --- | --- | --- |
| `--experimental-transform-types` | [nodejs/node#54283](https://github.com/nodejs/node/pull/54283) | v22.7.0, v23.0.0 | Node once had an opt-in transform mode for non-erasable syntax (`enum`, runtime namespaces). It is not the floor tsx benchmarks against. |
| `module.stripTypeScriptTypes()` API | [#55282](https://github.com/nodejs/node/pull/55282), [nodejs/node#54300](https://github.com/nodejs/node/issues/54300) | v22.13.0, v23.2.0 | Exposes type stripping as a public `node:module` API; useful for embedders, not a full loader. |
| TypeScript in `--eval` / STDIN | [#56359](https://github.com/nodejs/node/pull/56359) | v22.14.0, v23.6.0, v24.0.0 | Native TS isn't only file loading; the CLI eval/stdin path also strips types. |
| Unflag `--experimental-strip-types` | [#56350](https://github.com/nodejs/node/pull/56350), [nodejs/typescript#17](https://github.com/nodejs/typescript/issues/17) | v22.18.0, v23.6.0, v24.0.0 | This is tsx's `nativeTypeScript` gate: Node runs strip-only TS by default. |
| `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` | [#56610](https://github.com/nodejs/node/pull/56610) | v22.14.0, v23.7.0, v24.0.0 | Distinguishes unsupported non-erasable TS from invalid syntax. |
| Compile-cache integration | [#56629](https://github.com/nodejs/node/pull/56629), [nodejs/node#54741](https://github.com/nodejs/node/issues/54741) | v23.7.0, v24.0.0 | Native TS can cache stripped/transformed output; cold/warm comparisons need to account for Node's compile cache. |
| Recommend `erasableSyntaxOnly` | [#57271](https://github.com/nodejs/node/pull/57271) | v22.15.0, v23.10.0, v24.0.0 | Aligns TypeScript's checker with Node's strip-only runtime. |
| Remove experimental warning | [#58643](https://github.com/nodejs/node/pull/58643), [nodejs/typescript#24](https://github.com/nodejs/typescript/issues/24) | v22.18.0, v24.3.0 | Strip-only became quiet by default, but still release-candidate until stable. |
| Stable | [#60600](https://github.com/nodejs/node/pull/60600), [nodejs/typescript#24](https://github.com/nodejs/typescript/issues/24) | v24.12.0, v25.2.0 | Stable docs status; tsx's gate predates this because it tracks availability, not stability label. |
| Remove `--experimental-transform-types` | [#61803](https://github.com/nodejs/node/pull/61803), [nodejs/typescript#51](https://github.com/nodejs/typescript/issues/51) | v26.0.0 | Node 26+ is strip-only. Non-erasable transforms remain a gap tools such as tsx fill. |
| Type-stripped CJS sourceURL fix | [#63705](https://github.com/nodejs/node/pull/63705) | v26.4.0 | CJS type-stripped scripts now use `file:` URLs for sourceURL, aligning inspector behavior with ESM. |

## Runtime pipeline

- Node delegates stripping to Amaro: `amaro.transformSync` is loaded in [`typescript.js#L47-L48`](https://github.com/nodejs/node/blob/v24.0.0/lib/internal/modules/typescript.js#L47-L48). The public-ish internal API `stripTypeScriptTypes` starts at [`typescript.js#L101`](https://github.com/nodejs/node/blob/v24.0.0/lib/internal/modules/typescript.js#L101), and everything routes through `processTypeScriptCode` at [`typescript.js#L144`](https://github.com/nodejs/node/blob/v24.0.0/lib/internal/modules/typescript.js#L144).
- ESM format detection maps `.ts` to `module-typescript` / `commonjs-typescript` based on package type and syntax detection ([v24.0.0 `get_format.js#L133-L149`](https://github.com/nodejs/node/blob/v24.0.0/lib/internal/modules/esm/get_format.js#L133-L149)). In v26 the extension map is explicit for `.ts` / `.mts` / `.cts` ([`get_format.js#L31-L33`](https://github.com/nodejs/node/blob/v26.0.0/lib/internal/modules/esm/get_format.js#L31-L33)) before the same package/syntax logic later in the file ([`get_format.js#L189-L202`](https://github.com/nodejs/node/blob/v26.0.0/lib/internal/modules/esm/get_format.js#L189-L202)).
- The CommonJS loader strips TS formats before compile. v24.0.0 handles `module-typescript` / `commonjs-typescript` / `typescript` at [`loader.js#L1125-L1127`](https://github.com/nodejs/node/blob/v24.0.0/lib/internal/modules/cjs/loader.js#L1125-L1127), and v26 has the same strip-before-compile path via `stripTypeScriptModuleTypes` at [`loader.js#L182`](https://github.com/nodejs/node/blob/v26.0.0/lib/internal/modules/cjs/loader.js#L182) and the TS-format cases at [`loader.js#L1125-L1127`](https://github.com/nodejs/node/blob/v26.0.0/lib/internal/modules/cjs/loader.js#L1125-L1127).
- `--eval` and STDIN use the same internal TypeScript evaluator: v22.14.0 wires `evalTypeScript` into `eval_string` ([`eval_string.js#L39-L68`](https://github.com/nodejs/node/blob/v22.14.0/lib/internal/main/eval_string.js#L39-L68)) and `eval_stdin` ([`eval_stdin.js#L37-L47`](https://github.com/nodejs/node/blob/v22.14.0/lib/internal/main/eval_stdin.js#L37-L47)).
- Stripping runs after the module-hook chain, not before it. The ESM loader loads source through the hook chain first, then translates by the final format ([v24.15.0 `loader.js#L408-L414`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/loader.js#L408-L414)); the `commonjs-typescript` and `module-typescript` translators call `stripTypeScriptModuleTypes` on whatever source the hooks returned ([v24.15.0 `translators.js#L628-L643`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/translators.js#L628-L643)). A load hook that returns TypeScript source with a `-typescript` format gets native stripping; a hook that rewrites the format opts out of it.
- For `.ts` files in a package without a `type` field, the format follows the same contract as `.js`, including syntax detection. Detection happens at load time, not resolve time: `defaultResolve` reports a `null` format for these files, and the module system is determined when the source is read.
- tsx maps explicit `module-typescript` / `commonjs-typescript` resolve results directly to `module` / `commonjs` because Node already determined the module type ([v24.15.0 `get_format.js#L186-L207`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/esm/get_format.js#L186-L207)). When Node returns no format, tsx retains its package lookup and legacy CommonJS default, including for typeless `.ts` files.
- The transpiled output participates in the compile cache and V8 code cache keyed by source text and URL at the C++ layer ([v24.15.0 `module_wrap.cc#L518-L527`](https://github.com/nodejs/node/blob/v24.15.0/src/module_wrap.cc#L518-L527)), so any module compilation — including source produced by customization hooks — is cacheable when the compile cache is enabled.

## Public `stripTypeScriptTypes()` vs the internal loader path

`node:module` exposes stripping as a public API, but it is a different code path from what the loaders use, with different behavior:

| Aspect | Public `stripTypeScriptTypes()` | Internal `stripTypeScriptModuleTypes()` |
| --- | --- | --- |
| `node_modules` | No restriction | Refuses with `ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING` ([v24.15.0 `typescript.js#L180-L183`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/typescript.js#L180-L183)) |
| Compile cache | Not used | Keyed by filename via `getCompileCacheEntry` ([v24.15.0 `typescript.js#L198`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/typescript.js#L198)) |
| Mode | `'strip'` or `'transform'` accepted without any CLI flag through v25.x ([v24.15.0 `typescript.js#L112`](https://github.com/nodejs/node/blob/v24.15.0/lib/internal/modules/typescript.js#L112)); v26 accepts only `'strip'` ([v26.4.0 `typescript.js#L101`](https://github.com/nodejs/node/blob/v26.4.0/lib/internal/modules/typescript.js#L101)) | Follows `--experimental-transform-types` until its removal in v26 |
| Warning | Emits an `ExperimentalWarning` on first call, still present in v26 ([v26.4.0 `typescript.js#L92`](https://github.com/nodejs/node/blob/v26.4.0/lib/internal/modules/typescript.js#L92)) | None |

In strip mode, types are replaced with whitespace so line and column positions match the source and no source map is produced; `sourceMap: true` is only valid in transform mode. The docs warn that output should not be considered stable across Node versions ([v24.15.0 `module.md#L281`](https://github.com/nodejs/node/blob/v24.15.0/doc/api/module.md#L281)), so consumers should not key caches or assertions on exact output bytes.

Amaro reports two error classes, surfaced as distinct codes: `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` for valid TypeScript that requires transformation (`enum`, runtime `namespace`, parameter properties, `import =`/`export =` aliases), and `ERR_INVALID_TYPESCRIPT_SYNTAX` for source that does not parse. Decorators are neither: they are treated as JavaScript syntax and pass through stripping untouched, so they reach V8 as-is. TypeScript's `erasableSyntaxOnly` checker option flags exactly the unsupported-syntax set — not decorators or `accessor`, which are checker-legal JavaScript — so a project that type-checks under `erasableSyntaxOnly` never triggers the unsupported-syntax error at runtime.

## What Node intentionally does not do

The Node docs frame native TypeScript as a lightweight strip-only runtime, not a TypeScript compiler:

- Node ignores `tsconfig.json`; features depending on it, such as `paths` or downleveling, are unsupported ([v26.0.0 `typescript.md#L80-L88`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L80-L88)).
- File extensions are mandatory in imports and `require()` calls: `import './file.ts'`, not `import './file'` ([v26.0.0 `typescript.md#L128-L136`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L128-L136)).
- `.tsx` files are unsupported ([v26.0.0 `typescript.md#L128`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L128)). The open JSX issue is [nodejs/node#56822](https://github.com/nodejs/node/issues/56822).
- Non-erasable syntax errors: `enum`, runtime `namespace`, parameter properties, and import aliases ([v26.0.0 `typescript.md#L144-L152`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L144-L152)). In v26, `--experimental-transform-types` is gone, so this is no longer a runtime escape hatch ([v26.0.0 `typescript.md#L7`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L7), [nodejs/node#61803](https://github.com/nodejs/node/pull/61803)).
- TypeScript under `node_modules` is refused to discourage publishing TS-only packages ([v26.0.0 `typescript.md#L214-L217`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L214-L217)); the runtime throws `ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING` ([v26.0.0 `typescript.js#L159`](https://github.com/nodejs/node/blob/v26.0.0/lib/internal/modules/typescript.js#L159)). This is actively debated: [nodejs/node#58429](https://github.com/nodejs/node/issues/58429) (transpile TS in `node_modules`), [nodejs/node#58626](https://github.com/nodejs/node/issues/58626) (cross-package development), and open PR [nodejs/node#63853](https://github.com/nodejs/node/pull/63853) (allow type stripping in `node_modules`).
- TypeScript data URLs are not part of the normal file-loading story; [nodejs/node#61129](https://github.com/nodejs/node/issues/61129) tracked that gap.
- Extensionless TS executables are not supported; see [nodejs/node#59565](https://github.com/nodejs/node/issues/59565).
- TypeScript's CJS-only syntax (`import fs = require('fs')`, `export =`) is not erasable; see [nodejs/node#63977](https://github.com/nodejs/node/issues/63977).

## Recommended tsconfig contract

Node recommends TypeScript 5.8+ and a checker configuration that matches strip-only runtime behavior ([v24.12.0 `typescript.md#L80-L99`](https://github.com/nodejs/node/blob/v24.12.0/doc/api/typescript.md#L80-L99), [v26.0.0 `typescript.md#L93-L108`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L93-L108)):

```json
{
    "compilerOptions": {
        "noEmit": true,
        "target": "esnext",
        "module": "nodenext",
        "rewriteRelativeImportExtensions": true,
        "erasableSyntaxOnly": true,
        "verbatimModuleSyntax": true
    }
}
```

How those options relate to Node:

- `erasableSyntaxOnly` makes TypeScript reject non-erasable syntax before runtime. Node docs started recommending it after [nodejs/node#57271](https://github.com/nodejs/node/pull/57271); the TypeScript option exists for this exact runtime contract.
- `verbatimModuleSyntax` forces type-only imports to be written as `import type`, matching Node's runtime rule that it does not erase imports based on type information.
- `rewriteRelativeImportExtensions` is for projects that emit JavaScript; for type-check-only scripts that Node runs directly, `allowImportingTsExtensions` is the checker-side option that permits explicit `.ts` specifiers. Node's docs mention it because the runtime requires explicit `.ts` extensions ([v26.0.0 `typescript.md#L136-L139`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L136-L139)).
- `module: nodenext` is the closest checker model for Node's `.ts`/`.mts`/`.cts` module classification. Node does not transform module syntax across systems: ESM source needs ESM syntax, and CJS source needs `require`/`module.exports` ([v26.0.0 `typescript.md#L110-L126`](https://github.com/nodejs/node/blob/v26.0.0/doc/api/typescript.md#L110-L126)).

## Relationship to tsx

Native stripping covers the common, erasable-syntax case. That is the ideal path for projects that fit it: less tooling, less startup overhead, and fewer compatibility layers. tsx remains useful for the gaps Node intentionally leaves to tools:

- tsx transforms syntax Node rejects or no longer supports transforming (`enum`, parameter properties, JSX/TSX, decorators, CJS TypeScript syntax, import aliases);
- tsx implements extension and path behavior Node explicitly leaves to tools (`paths`, extensionless imports, `.js` specifiers that map to `.ts`, TypeScript under dependency graphs when needed);
- tsx handles CJS/ESM interop and named exports around transformed source (see [`cjs-esm-interop.md`](./cjs-esm-interop.md));
- tsx's benchmark `native-ts` scenario measures the runtime floor Node provides, while `esm-ts` measures the extra transform and resolution surface tsx adds for those gaps.
- Module-system classification differs for one population: for `.ts` files in a package without a `type` field, Node applies the same syntax detection it applies to `.js`, while tsx's format detection predates detect-module and classifies those files as CommonJS. Code relying on ESM-only semantics (such as top-level await) in a typeless package runs under plain `node` but is converted to CommonJS by tsx.

## Remaining gaps Node is closing

- `node_modules` restriction: [#58429](https://github.com/nodejs/node/issues/58429), [#58626](https://github.com/nodejs/node/issues/58626), [#63853](https://github.com/nodejs/node/pull/63853). If Node allows TS in workspace dependencies, tsx can re-audit how much dependency transform/resolution support users still need.
- JSX / `.tsx`: [#56322](https://github.com/nodejs/node/issues/56322), [#56822](https://github.com/nodejs/node/issues/56822). If Node accepts `.tsx`, tsx's native-floor benchmark changes substantially and another major gap closes.
- Ecosystem adaptation after default-on stripping: [#59364](https://github.com/nodejs/node/issues/59364). Important when deciding whether to widen tsx's native fast paths.
- Source identity: [#63705](https://github.com/nodejs/node/pull/63705) uses `file:` URLs for type-stripped CJS `sourceURL`; tsx should keep its own data-URL/sourceURL behavior aligned with inspector expectations.
