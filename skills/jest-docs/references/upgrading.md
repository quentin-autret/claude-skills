# Upgrading to Jest 30 and migrating to Jest

## Contents

- Jest 29 to 30: compatibility
- Removed alias matchers
- Matcher and type changes
- CLI and config changes
- Mock API changes
- Runner and module changes
- Snapshot changes
- Older changes still met in code (Jest 29)
- Migrating from another runner

## Jest 29 to 30: compatibility

- Minimum Node is 18 (14, 16, 19 and 21 dropped). Minimum TypeScript is 5.4 with Jest types. https://jestjs.io/docs/upgrading-to-jest30#compatibility
- `jest-environment-jsdom` moves to JSDOM 26, which may change DOM behavior or add new warnings. https://jestjs.io/docs/upgrading-to-jest30#compatibility
- Set `errorOnDeprecated: true` to surface deprecated calls while upgrading. https://jestjs.io/docs/configuration#errorondeprecated-boolean

## Removed alias matchers

Use the canonical names. The `eslint-plugin-jest` rule `no-alias-methods` fixes them. https://jestjs.io/docs/upgrading-to-jest30#removal-of-alias-matcher-functions

| Removed            | Use                        |
| ------------------ | -------------------------- |
| `toBeCalled`       | `toHaveBeenCalled`         |
| `toBeCalledTimes`  | `toHaveBeenCalledTimes`    |
| `toBeCalledWith`   | `toHaveBeenCalledWith`     |
| `lastCalledWith`   | `toHaveBeenLastCalledWith` |
| `nthCalledWith`    | `toHaveBeenNthCalledWith`  |
| `toReturn`         | `toHaveReturned`           |
| `toReturnTimes`    | `toHaveReturnedTimes`      |
| `toReturnWith`     | `toHaveReturnedWith`       |
| `lastReturnedWith` | `toHaveLastReturnedWith`   |
| `nthReturnedWith`  | `toHaveNthReturnedWith`    |
| `toThrowError`     | `toThrow`                  |

## Matcher and type changes

- Non-enumerable properties are excluded from object matchers, which can change `expect.objectContaining` and equality results. https://jestjs.io/docs/upgrading-to-jest30#non-enumerable-properties
- `toHaveBeenCalledWith` and the other `CalledWith` matchers infer parameter types: a wrong argument type is now a compile error. Fix the argument or cast. https://jestjs.io/docs/upgrading-to-jest30#improved-type-inference-for-calledwith

## CLI and config changes

- `--testPathPattern` becomes `--testPathPatterns`. https://jestjs.io/docs/upgrading-to-jest30#--testpathpattern-was-renamed-to---testpathpatterns

```bash
# Jest 29
jest --testPathPattern="unit/.*"
# Jest 30
jest --testPathPatterns "unit/.*" "integration/.*"
```

https://jestjs.io/docs/upgrading-to-jest30#--testpathpattern-was-renamed-to---testpathpatterns

- `jest --init` is removed: use `npm init jest@latest`. https://jestjs.io/docs/upgrading-to-jest30#removed---init-command
- Flags requiring a value throw when it is missing (for example a bare `--maxWorkers`). Check npm scripts. https://jestjs.io/docs/upgrading-to-jest30#other-cli-changes
- A `--filter` module returns `{filtered: Array<string>}`, not an array. https://jestjs.io/docs/upgrading-to-jest30#other-cli-changes
- `.mts` and `.cts` join `moduleFileExtensions`, and `testMatch` picks up `.mjs`, `.cjs`, `.mts` and `.cts` tests. Non-test files with these extensions may now match. https://jestjs.io/docs/upgrading-to-jest30#support-for-mts-and-cts-file-extensions
- `glob` 10 is stricter on brace expansion and extglobs: fix `testMatch` or other globs that stop matching. https://jestjs.io/docs/upgrading-to-jest30#glob-pattern-matching-changes

## Mock API changes

- `jest.genMockFromModule` is removed: use `jest.createMockFromModule` (same behavior). https://jestjs.io/docs/upgrading-to-jest30#jestgenmockfrommodule-removed
- Types `MockFunctionMetadata`, `MockFunctionMetadataType` and `SpyInstance` are removed. Replace `jest.SpyInstance` with `jest.Spied`. https://jestjs.io/docs/upgrading-to-jest30#removed-mock-function-types
- `jest.mock()` paths are case-sensitive: `jest.mock('./path/to/FILENAME.js')` no longer finds `filename.js`. https://jestjs.io/docs/upgrading-to-jest30#jestmock-only-works-with-case-sensitive-module-path

## Runner and module changes

- Unhandled rejections wait one extra event loop turn before failing, so late-caught rejections no longer fail tests. `waitForUnhandledRejections` controls it. Most users need not change the default. https://jestjs.io/docs/upgrading-to-jest30#unhandled-promise-rejections-in-tests
- Custom `TestSequencer` subclasses must adapt: the API exposes `globalConfig` and `contexts`. https://jestjs.io/docs/upgrading-to-jest30#custom-test-sequencers
- A programmatic `Runtime` needs a `globalConfig` argument. https://jestjs.io/docs/upgrading-to-jest30#required-globalconfig-in-runtime
- Internals are bundled and packages declare `"exports"`: deep imports such as `require('jest-runner/build/testWorker')` break. Use public APIs only. https://jestjs.io/docs/upgrading-to-jest30#esm-module-support-and-internal-restructuring

## Snapshot changes

- Snapshot headers use full URLs instead of goo.gl links, error snapshots include `cause`, the React serializer drops empty-string children, and `pretty-format` prints `ArrayBuffer` and `DataView` readably. Review, then update snapshots. https://jestjs.io/docs/upgrading-to-jest30#snapshots-and-output-changes

## Older changes still met in code (Jest 29)

- Default `snapshotFormat` is `{escapeString: false, printBasicPrototype: false}`. The old look is `{escapeString: true, printBasicPrototype: true}`. https://jestjs.io/docs/upgrading-to-jest29#snapshot-format
- `jest.mocked()` wraps deeply: drop the `true` second argument and use `{shallow: true}` for one level. https://jestjs.io/docs/upgrading-to-jest29#jestmocked

```diff
- const mockedObject = jest.mocked(someObject, true);
+ const mockedObject = jest.mocked(someObject);
```

https://jestjs.io/docs/upgrading-to-jest29#jestmocked

- `MaybeMockedDeep` is `Mocked`, `MaybeMocked` is `MockedShallow`. https://jestjs.io/docs/upgrading-to-jest29#jest-mock
- The `ConvertAnsi` plugin of `pretty-format` is replaced by `jest-serializer-ansi-escapes`. https://jestjs.io/docs/upgrading-to-jest29#pretty-format
- `extraGlobals` was renamed `sandboxInjectedGlobals` in Jest 28. https://jestjs.io/docs/configuration#sandboxinjectedglobals-arraystring

## Migrating from another runner

- Jasmine and Mocha style APIs are mostly compatible. https://jestjs.io/docs/migration-guide
- `npx jest-codemods` (third party) migrates AVA, Chai, Expect.js, Jasmine, Mocha, proxyquire, Should.js, Tape and Sinon. https://jestjs.io/docs/migration-guide#jest-codemods
