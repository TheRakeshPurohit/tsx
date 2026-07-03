# Node.js internals notes

Maintainer notes tracking how Node module-loading internals affect tsx. These are not user docs (`files: ["dist"]` keeps them out of the npm package).

tsx fills gaps in Node's TypeScript and module-loading story. As Node adds native support — type stripping, `require(esm)`, sync module hooks — those gaps get smaller, which is the preferred direction. These notes keep tsx aligned with Node behavior and identify the remaining gaps where tsx still helps users today.

## Reading order

### Mechanism stories

| File | Covers |
| --- | --- |
| [module-hooks.md](./module-hooks.md) | Async `module.register()` vs sync `module.registerHooks()`, loader-worker cost, CJS-reload safety |
| [cjs-loader.md](./cjs-loader.md) | CJS resolution, cache identity, `require.extensions`, eager ESM error decoration |
| [cjs-esm-interop.md](./cjs-esm-interop.md) | ESM importing CJS, CJS requiring ESM, named-export preparsing, `module.exports` interop |
| [type-stripping.md](./type-stripping.md) | Node's native TypeScript pipeline, limitations, current issues, tsconfig contract |

### Reference maps

| File | Covers |
| --- | --- |
| [gate-reference.md](./gate-reference.md) | Smaller gates: import attributes, `import.meta` props, wasm, package main, test runner flags |
| [node-integration-points.md](./node-integration-points.md) | Node loader integration points to re-verify across releases |

## Gate index

| `src/utils/node-features.ts` gate | Context |
| --- | --- |
| `moduleRegister` | [module-hooks.md](./module-hooks.md#moduleregister--async-hooks-loader-worker-thread) |
| `moduleRegisterHooksCjsReload` | [module-hooks.md](./module-hooks.md#cjs-reload-safety--the-boundary-tsx-actually-gates-on) |
| `importAttributes` | [gate-reference.md](./gate-reference.md#import-attributes-importattributes) |
| `testRunnerGlob` | [gate-reference.md](./gate-reference.md#test-runner-glob-testrunnerglob) |
| `cliTestFlag` | [gate-reference.md](./gate-reference.md#--test-flag-clitestflag) |
| `esmLoadReadFile` | [cjs-esm-interop.md](./cjs-esm-interop.md#esm-importing-cjs) |
| `importMetaPathProperties` | [gate-reference.md](./gate-reference.md#importmeta-path-properties-importmetapathproperties) |
| `requireEsm` | [cjs-esm-interop.md](./cjs-esm-interop.md#cjs-requiring-esm) |
| `requireEsmNoWarning` | [cjs-esm-interop.md](./cjs-esm-interop.md#cjs-requiring-esm) |
| `cjsNamespaceModuleExports` | [cjs-esm-interop.md](./cjs-esm-interop.md#cjs-requiring-esm) |
| `nativeTypeScript` | [type-stripping.md](./type-stripping.md#timeline) |
| `wasmModules` | [gate-reference.md](./gate-reference.md#wasm-modules-wasmmodules) |
| `cjsNamespaceFromLoadHook` | [cjs-esm-interop.md](./cjs-esm-interop.md#esm-importing-cjs) |
| `requireEsmExtensionlessMjs` | [cjs-esm-interop.md](./cjs-esm-interop.md#cjs-requiring-esm) |
| `modulePackageMainResolution` | [gate-reference.md](./gate-reference.md#legacymainresolve-assertion-fix-modulepackagemainresolution) |

## Entry format

Each documented behavior should record:

1. what changed in Node;
2. the Node PR or issue;
3. verified versions — first release per major line, confirmed from Node git history;
4. exact tagged Node source anchors (`github.com/nodejs/node/blob/<tag>/...#L...`), ideally last-without and first-with for boundaries;
5. tsx code paths affected;
6. coverage when relevant.

Optional sections:

- `## Decisions` for untestable tradeoffs and rejected alternatives;
- `## Implementation history in tsx` for old commits/PRs that explain why the current approach exists.

Testable behavior belongs in `tests/`; these notes explain context and maintenance intent.

## Verification workflow

Version boundaries are pinned from Node's git history, because backports often land after the main-line commit.

```text
cd /path/to/nodejs/node
git fetch --tags

# Every commit carrying a PR-URL trailer = main-line commit + all backport
# cherry-picks. Collect all release tags containing them, first per major line:
pr=59929
{ for sha in $(git log --grep "PR-URL: https://github.com/nodejs/node/pull/${pr}\$" --format=%H --all); do
    git tag --contains "$sha" | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$'
  done; } | sort -u -V | awk -F. '{maj=$1} maj!=prev{print; prev=maj}'
```

After finding the boundary, verify the exact source shape with `git show <tag>:<path>` and link the public GitHub URL. Do not link to moving branches for source claims.

Verification caveats:

- A PR can be the **fix** to a feature, not its **introduction**. When PR-contains and source-reading disagree, read the source at the tag and document both.
- A fix backported only to an older line means newer lines may never have had the bug; model that as a range or a line-specific gate.
- Some gates describe a bug window `[from, before)` — opened by one PR, closed by another.
