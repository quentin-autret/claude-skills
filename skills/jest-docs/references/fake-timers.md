# Fake timers (Jest 30.5)

## Contents

- Enabling and restoring
- What gets faked
- Running timers
- Promises mixed with timers
- Clock control
- Configuration
- Legacy fake timers

## Enabling and restoring

- `jest.useFakeTimers()` is backed by `@sinonjs/fake-timers`. It is global for the file until `jest.useRealTimers()`. Calling it again resets internal state with the new options. https://jestjs.io/docs/jest-object#jestusefaketimersfaketimersconfig
- Restore in `afterEach`: https://jestjs.io/docs/jest-object#jestuserealtimers

```js
beforeEach(() => {
  jest.useFakeTimers();
});
afterEach(() => {
  jest.useRealTimers();
});
```

- Assert on timer calls with `jest.spyOn(global, 'setTimeout')` after `jest.useFakeTimers()`. https://jestjs.io/docs/timer-mocks#enable-fake-timers

## What gets faked

- Faked: `Date`, `performance.now()`, `queueMicrotask`, `setImmediate`, `setInterval`, `setTimeout` and their clear functions. In Node also `process.hrtime` and `process.nextTick`. In jsdom also `requestAnimationFrame` and `requestIdleCallback`. https://jestjs.io/docs/jest-object#jestusefaketimersfaketimersconfig
- Leave an API real with `doNotFake`. The `jest.useFakeTimers()` list accepts `'Temporal'`. https://jestjs.io/docs/jest-object#jestusefaketimersfaketimersconfig

```js
jest.useFakeTimers({doNotFake: ['performance']});
```

https://jestjs.io/docs/timer-mocks#selective-faking

## Running timers

- `jest.advanceTimersByTime(ms)` runs timers due within the window, including ones scheduled inside it. It also accepts a `Temporal.Duration`, calendar units (years, months, weeks) throw. https://jestjs.io/docs/jest-object#jestadvancetimersbytimemstorun
- `jest.runOnlyPendingTimers()` runs only the timers pending now. It is the answer for recursive timers. https://jestjs.io/docs/timer-mocks#run-pending-timers
- `jest.runAllTimers()` loops until queues are empty. On recursive timers it aborts with "Aborting after running 100000 timers, assuming an infinite loop!". `timerLimit` changes the limit. https://jestjs.io/docs/timer-mocks#run-pending-timers
- `jest.runAllTicks()` drains the `nextTick` queue. https://jestjs.io/docs/jest-object#jestrunallticks
- `jest.advanceTimersToNextFrame()` runs `requestAnimationFrame` callbacks. Frames fire every 16 ms, so after 6 ms it advances 10 ms. https://jestjs.io/docs/timer-mocks#advance-timers-to-the-next-frame
- `jest.getTimerCount()` counts pending timers, `jest.clearAllTimers()` drops them. https://jestjs.io/docs/jest-object#jestgettimercount

## Promises mixed with timers

- Use the async variants when code awaits a promise between timers: `jest.advanceTimersByTimeAsync`, `jest.runAllTimersAsync`, `jest.runOnlyPendingTimersAsync`, `jest.advanceTimersToNextTimerAsync`. They let scheduled promise callbacks run before the timers. https://jestjs.io/docs/jest-object#jestrunalltimersasync

## Clock control

- `jest.setSystemTime(now)` moves the clock without firing timers. With `'Temporal'` faked, `Temporal.Now.*` follows the fake clock but `Temporal.Now.timeZoneId()` does not. https://jestjs.io/docs/jest-object#jestsetsystemtimenow-number--date--temporalinstant--temporalzoneddatetime
- `jest.getRealSystemTime()` returns real time while `Date` is faked. `jest.now()` returns the current clock time in ms. https://jestjs.io/docs/jest-object#jestgetrealsystemtime
- `jest.setTimerTickMode({mode})` takes `manual`, `nextAsync` or `interval` (delta default 20). `interval` equals `advanceTimers: true`. https://jestjs.io/docs/jest-object#jestsettimertickmodemode

## Configuration

- Options of `jest.useFakeTimers()` and of the `fakeTimers` config: `advanceTimers` (boolean or ms, default `false`, `true` auto-advances 20 ms every 20 ms), `doNotFake` (default `[]`), `now` (default `Date.now()`), `timerLimit` (default `100_000`), `legacyFakeTimers` (default `false`). https://jestjs.io/docs/configuration#faketimers-object
- The `fakeTimers` config is the default for `jest.useFakeTimers()`, an object passed in a test overrides it. `enableGlobally: true` fakes timers in every test with no call. https://jestjs.io/docs/configuration#faketimers-object

```js
fakeTimers: {enableGlobally: true, doNotFake: ['nextTick'], timerLimit: 1000},
```

https://jestjs.io/docs/configuration#faketimers-object

## Legacy fake timers

- `legacyFakeTimers: true` takes no other option and swaps timers for Jest mock functions. https://jestjs.io/docs/jest-object#jestusefaketimersfaketimersconfig
- Missing in legacy mode: the `*Async` variants, `setSystemTime`, `advanceTimersToNextFrame`, `setTimerTickMode`, `getRealSystemTime`. `runAllImmediates` exists only in legacy mode. https://jestjs.io/docs/jest-object#jestusefaketimersfaketimersconfig
