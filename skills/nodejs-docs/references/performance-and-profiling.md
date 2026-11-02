# perf_hooks and node:v8

## Contents

- User timing and observers
- Event loop health
- Histograms
- Heap snapshots and queries
- CPU and heap profiles
- Other v8 APIs
- Old patterns

## User timing and observers

- Clear marks and measures after observing: they stay in the global timeline until `clearMarks()` / `clearMeasures()`. https://nodejs.org/docs/latest/api/perf_hooks.html#performancemarkname-options

```js
const obs = new PerformanceObserver((items) => {
  console.log(items.getEntries()[0].duration);
  performance.clearMarks();
});
obs.observe({ type: 'measure' });
performance.mark('A');
performance.measure('A to Now', 'A');
```
https://nodejs.org/docs/latest/api/perf_hooks.html

- Disconnect observers once unused: each adds overhead. https://nodejs.org/docs/latest/api/perf_hooks.html#new-performanceobservercallback
- Pass `type` or `entryTypes`, never both. `buffered: true` replays earlier entries. https://nodejs.org/docs/latest/api/perf_hooks.html#performanceobserverobserveoptions
- Call `mark()`/`measure()` on `performance` itself, not detached, and name them. `measure()` throws for a missing mark. https://nodejs.org/docs/latest/api/perf_hooks.html#performancemeasurename-startmarkoroptions-endmark
- The resource timing buffer holds 250 entries: handle `'resourcetimingbufferfull'`. https://nodejs.org/docs/latest/api/perf_hooks.html#performancesetresourcetimingbuffersizemaxsize
- `timerify(fn)` reports to observers of type `'function'`. Async functions are timed until their `finally`. https://nodejs.org/docs/latest/api/perf_hooks.html#perf_hookstimerifyfn-options

## Event loop health

- Event loop utilization is time spent outside the event provider, not CPU: a process blocked in `spawnSync('sleep')` reports 1. Pass the previous result to get a delta. `perf_hooks.eventLoopUtilization()` is top-level since v25.2.0 (backported to v24.12.0). https://nodejs.org/docs/latest/api/perf_hooks.html#perf_hookseventlooputilizationutilization1-utilization2

```js
const elu = eventLoopUtilization();
// ... later
console.log(eventLoopUtilization(elu).utilization);
```

- `monitorEventLoopDelay({ resolution: 10 })` returns a nanosecond histogram: `enable()`, later `disable()`, then read `percentile(99)`. https://nodejs.org/docs/latest/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions
- `samplePerIteration: true` (added in v26.5.0) samples once per loop turn and does not keep the loop alive. Never compare its numbers with interval sampling. https://nodejs.org/docs/latest/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions

## Histograms

- `createHistogram({ lowest, highest, figures })`: figures 1 to 5, `highest` at least twice `lowest`. https://nodejs.org/docs/latest/api/perf_hooks.html#perf_hookscreatehistogramoptions
- v26 additions (not on v24): `halfLife` and `threshold` options with `burnRate(sloTarget)` (above 1 means the error budget is depleting), statistics such as `cdf`, `percentileCI`, `mannWhitneyTest` (v26.8.0), `export()` / `importHistogram()` (v26.9.0), `createSlidingWindowHistogram()` (v26.10.0), `diff()` (v26.11.0). https://nodejs.org/docs/latest/api/perf_hooks.html#class-histogram , https://nodejs.org/docs/latest/api/perf_hooks.html#perf_hookscreateslidingwindowhistogramoptions

```js
const h = createHistogram({ halfLife: 100, threshold: 200_000_000 });
if (h.burnRate(0.999) > 1) { /* budget depleting */ }
```

## Heap snapshots and queries

- A heap snapshot needs about twice the current heap in memory and blocks the loop: the OOM killer may end the process. A snapshot covers one isolate, so each worker writes its own. https://nodejs.org/docs/latest/api/v8.html#v8writeheapsnapshotfilenameoptions
- `v8.setHeapSnapshotNearHeapLimit(limit)` (stable) is a no-op if the CLI flag is set or when called twice. https://nodejs.org/docs/latest/api/v8.html#v8setheapsnapshotnearheaplimitlimit
- Write memory-leak regression tests with `v8.queryObjects(Ctor, { format: 'count' })` (stable since v25.4.0, backported to v24.13.1). It runs a full GC first, and subclasses count too. https://nodejs.org/docs/latest/api/v8.html#v8queryobjectsctor-options
- The snapshot JSON schema is undocumented and V8-specific. https://nodejs.org/docs/latest/api/v8.html#v8writeheapsnapshotfilenameoptions

## CPU and heap profiles

- `v8.startCpuProfile()` (added v25.0.0, backported to v24.12.0) returns a handle whose `stop()` gives the profile and which supports `using`. https://nodejs.org/docs/latest/api/v8.html#v8startcpuprofileoptions

```js
const handle = v8.startCpuProfile({ sampleInterval: 1, maxBufferSize: 10_000 });
const profile = handle.stop();
```

- `v8.startHeapProfile()` (added in v26.1.0) records a sampling heap profile. `v8.setHeapProfileNearHeapLimit()` (added in v26.9.0, experimental) writes it near OOM, but only if a profile was started. https://nodejs.org/docs/latest/api/v8.html#v8startheapprofileoptions , https://nodejs.org/docs/latest/api/v8.html#v8setheapprofilenearheaplimitlimit
- `v8.GCProfiler` reports GC cost for the current thread. https://nodejs.org/docs/latest/api/v8.html#class-v8gcprofiler

## Other v8 APIs

- `v8.setFlagsFromString()` after startup can crash, lose data or do nothing. https://nodejs.org/docs/latest/api/v8.html#v8setflagsfromstringflags
- `v8.serialize()` output is backward compatible but equal values may produce different bytes. https://nodejs.org/docs/latest/api/v8.html#serialization-api
- `v8.takeCoverage()` writes `NODE_V8_COVERAGE` data on demand. `stopCoverage()` releases counters. https://nodejs.org/docs/latest/api/v8.html#v8takecoverage
- Startup snapshots (`--build-snapshot`) are stable since v24.0.0. https://nodejs.org/docs/latest/api/v8.html#startup-snapshot-api

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `entry.kind`, `entry.flags` | `entry.detail.kind`, `entry.detail.flags` | https://nodejs.org/docs/latest/api/perf_hooks.html#performancenodeentrydetail |
| destructured `const { mark } = performance` | `performance.mark()` | https://nodejs.org/docs/latest/api/perf_hooks.html#performancemarkname-options |
| `v8.promiseHooks` for request context | `AsyncLocalStorage` | https://nodejs.org/docs/latest/api/v8.html#promise-hooks |
