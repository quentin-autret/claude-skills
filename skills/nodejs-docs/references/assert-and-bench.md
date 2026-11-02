# node:assert and node:bench

## Contents

- Strict mode
- throws and rejects
- Deep equality
- node:bench (added in v26.9.0)
- Old patterns

## Strict mode

- Import `node:assert/strict`: the loose names behave like the strict ones and failures show diffs. `NO_COLOR` removes diff colors. https://nodejs.org/docs/latest/api/assert.html#strict-assertion-mode

```js
import assert from 'node:assert/strict';
```

- The legacy loose functions use `==`: `assert.deepEqual(/a/gi, new Date())` passes. https://nodejs.org/docs/latest/api/assert.html#legacy-assertion-mode
- `message` can be a string, an `Error` (thrown as is) or `(actual, expected) => string` called only on failure. https://nodejs.org/docs/latest/api/assert.html#message-parameter-semantics
- Configure with `new assert.Assert({ diff: 'full', strict: true })` (added v24.6.0) and call methods on the instance: destructuring them loses the options. https://nodejs.org/docs/latest/api/assert.html#new-assertassertoptions

## throws and rejects

- Never pass a string as the second argument of `assert.throws()`: if it equals the message Node throws `ERR_AMBIGUOUS_ARGUMENT`, otherwise it is only the failure message. Pass a RegExp, a class or a validation object. https://nodejs.org/docs/latest/api/assert.html#assertthrowsfn-error-message

```js
assert.throws(() => { throw err; }, { name: 'TypeError', message: /Wrong/ });
```

- In a validation object only listed properties are compared. RegExps match string properties at the top level only. A validation function must return `true`. https://nodejs.org/docs/latest/api/assert.html#assertthrowsfn-error-message
- Always `await assert.rejects()`. If the function throws synchronously or returns a non-promise, the matcher is skipped. https://nodejs.org/docs/latest/api/assert.html#assertrejectsasyncfn-error-message
- Skip `doesNotThrow()` / `doesNotReject()`: they add nothing over running the code. https://nodejs.org/docs/latest/api/assert.html#assertdoesnotthrowfn-error-message

## Deep equality

- `deepStrictEqual` compares with `Object.is`, checks prototypes, Error `name`, `message`, `cause`, and treats Map and Set as unordered. `WeakMap`, `WeakSet` and (since v25.0.0) `Promise` are equal only to themselves. https://nodejs.org/docs/latest/api/assert.html#comparison-details_1
- Compare a subset with `assert.partialDeepStrictEqual(actual, expected)` (stable since v24.0.0): only properties in `expected` count and prototypes are ignored. https://nodejs.org/docs/latest/api/assert.html#assertpartialdeepstrictequalactual-expected-message

## node:bench (added in v26.9.0)

Early development (1.0), not on v24. Needs `--experimental-bench` and the `node:` scheme. https://nodejs.org/docs/latest/api/bench.html

- Each sample gets a fresh context that must call `b.start()` and `b.end(operations)` once, or `b.record(sample)` once. Work outside is not measured. Defaults: 30 samples, 10 warmup. https://nodejs.org/docs/latest/api/bench.html#measurement-model

```js
import { bench, suite } from 'node:bench';
suite('URL', () => {
  bench('construct', (b) => {
    const operations = 10_000; let total = 0;
    b.start();
    for (let i = 0; i < operations; i++) total += new URL('https://a.org').href.length;
    b.end(operations);
    if (total === 0) throw new Error('Unexpected URL result');
  });
});
// node --experimental-bench --bench benchmark.mjs
```

- Make results observable (validate an aggregate) or the JIT may remove the work. Run enough operations per sample and inspect raw samples for warmup, GC and multimodality. https://nodejs.org/docs/latest/api/bench.html#measurement-integrity
- `summary.mean` is the mean of per-sample rates, not pooled throughput. There is no built-in baseline comparison: compare raw samples with your own statistics. Do not use a confidence interval as a pass/fail gate. https://nodejs.org/docs/latest/api/bench.html#comparing-benchmark-results , https://nodejs.org/docs/latest/api/bench.html#measurement-integrity
- With `--bench`, files must not call `run()`, and preloads must not declare benchmarks. Timeouts cannot interrupt synchronous code. https://nodejs.org/docs/latest/api/bench.html#command-line-runner , https://nodejs.org/docs/latest/api/bench.html#benchname-options-fn

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `assert.equal`, `assert.deepEqual` (legacy) | `strictEqual`, `deepStrictEqual` | https://nodejs.org/docs/latest/api/assert.html#assertdeepequalactual-expected-message |
| `assert.throws(fn, 'message')` | `assert.throws(fn, { message: '...' })` | https://nodejs.org/docs/latest/api/assert.html#assertthrowsfn-error-message |
| `assert.CallTracker` | removed (End-of-Life) | https://nodejs.org/docs/latest/api/deprecations.html#DEP0173 |
