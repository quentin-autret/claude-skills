# node:worker_threads and node:cluster

## Contents

- When to use workers
- Creating workers
- Messages, cloning and transfer
- Version-flagged worker APIs
- cluster
- Old patterns

## When to use workers

- Workers help CPU-heavy JavaScript, not I/O: built-in async I/O is more efficient. Unlike `child_process` or `cluster`, workers can share memory. https://nodejs.org/docs/latest/api/worker_threads.html#worker-threads , https://nodejs.org/docs/latest/api/cluster.html
- Use a pool of workers: creating one per task costs more than it gains. In a pool, correlate tasks with `AsyncResource` for diagnostics. https://nodejs.org/docs/latest/api/worker_threads.html#worker-threads

## Creating workers

- Handle `'error'` and a non-zero `'exit'`: https://nodejs.org/docs/latest/api/worker_threads.html#event-error

```js
const worker = new Worker(__filename, { workerData: script });
worker.on('message', resolve);
worker.once('error', reject);
worker.once('exit', (code) => { if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`)); });
```

- `await using worker = new Worker(...)` terminates it at scope exit (added v24.2.0). `terminate()` returns a promise. https://nodejs.org/docs/latest/api/worker_threads.html#workersymbolasyncdispose , https://nodejs.org/docs/latest/api/worker_threads.html#workerterminate
- Workers inherit CLI flags. An explicit `execArgv`, even `[]`, replaces them and can drop `--permission` grants. V8 flags (such as `--max-old-space-size`) are not supported there; `resourceLimits` sets heap limits. https://nodejs.org/docs/latest/api/worker_threads.html#new-workerfilename-options
- A worker started unconditionally from a `-r` preload recurses until crash (workers inherit preloads) unless `execArgv` is set. https://nodejs.org/docs/latest/api/worker_threads.html#launching-worker-threads-from-preload-scripts
- `resourceLimits` cover only the JS heap, not ArrayBuffers. `stackSizeMb` defaults to 4. https://nodejs.org/docs/latest/api/worker_threads.html#new-workerfilename-options
- Relative `filename` must start with `./` or `../` and resolves against the cwd. https://nodejs.org/docs/latest/api/worker_threads.html#new-workerfilename-options
- Worker stdio goes through messages: synchronous code on the main thread can hold it back. https://nodejs.org/docs/latest/api/worker_threads.html#synchronous-blocking-of-stdio
- Each worker gets a copy of `process.env`. `setEnvironmentData()` seeds new workers. https://nodejs.org/docs/latest/api/worker_threads.html#worker_threadssetenvironmentdatakey-value

## Messages, cloning and transfer

- Messages use structured clone: prototypes, classes, accessors and non-enumerable properties are lost (a `Buffer` arrives as `Uint8Array`, `URL` cannot be cloned). https://nodejs.org/docs/latest/api/worker_threads.html#considerations-when-cloning-objects-with-prototypes-classes-and-accessors
- An `ArrayBuffer` not in `transferList` is copied. Transferring detaches it and every view on it. A `SharedArrayBuffer` is shared and cannot be transferred. https://nodejs.org/docs/latest/api/worker_threads.html#considerations-when-transferring-typedarrays-and-buffers

```js
port.postMessage(uint8Array);                      // copy
port.postMessage(uint8Array, [uint8Array.buffer]); // transfer, uint8Array now empty
```

- Pooled Buffers (`Buffer.from()`, `Buffer.allocUnsafe()`) cannot be transferred: the whole pool is cloned. Use `Buffer.alloc()` for data you transfer, and `markAsUntransferable()` to force cloning of an `ArrayBuffer`. https://nodejs.org/docs/latest/api/worker_threads.html#considerations-when-transferring-typedarrays-and-buffers
- Cloneable native objects include `KeyObject`, `CryptoKey`, `FileHandle`, `X509Certificate`, `MessagePort`, `net.BlockList`, and TCP `net.Socket`/`net.Server` in the transfer list. A transferred socket must be fresh and not yet reading. https://nodejs.org/docs/latest/api/worker_threads.html#portpostmessagevalue-transferlist
- For threads that are not parent and child, use `postMessageToThread()` (active development). https://nodejs.org/docs/latest/api/worker_threads.html#worker_threadspostmessagetothreadthreadid-value-transferlist-timeout

## Version-flagged worker APIs

| API | Added | Source |
| :-- | :-- | :-- |
| `worker_threads.locks` (Web Locks) | v24.5.0, experimental | https://nodejs.org/docs/latest/api/worker_threads.html#worker_threadslocks |
| `threadName` | v24.6.0 | https://nodejs.org/docs/latest/api/worker_threads.html#worker_threadsthreadname |
| `worker.startCpuProfile()`, `startHeapProfile()` | v24.8.0, v24.9.0 | https://nodejs.org/docs/latest/api/worker_threads.html#workerstartcpuprofileoptions |
| `isInternalThread` | v23.7.0 | https://nodejs.org/docs/latest/api/worker_threads.html#worker_threadsisinternalthread |

## cluster

- Workers are `fork()`ed processes. The default scheduling is round-robin except on Windows. The OS-distributed alternative is unbalanced in practice. Set `cluster.schedulingPolicy` or `NODE_CLUSTER_SCHED_POLICY` before the first fork. https://nodejs.org/docs/latest/api/cluster.html#how-it-works , https://nodejs.org/docs/latest/api/cluster.html#clusterschedulingpolicy
- Node provides no routing logic: do not rely too heavily on in-memory state (sessions, login) across workers. Node does not manage the worker count: respawn yourself. https://nodejs.org/docs/latest/api/cluster.html#how-it-works
- `listen(0)` gives every worker the same random port. Derive a port from the worker id if each needs its own. https://nodejs.org/docs/latest/api/cluster.html#how-it-works
- `worker.disconnect()` waits for servers to close but not for client connections: tell them to close and add a timeout that calls `worker.kill()`. https://nodejs.org/docs/latest/api/cluster.html#workerdisconnect
- Respawn only accidental exits by checking `worker.exitedAfterDisconnect`: https://nodejs.org/docs/latest/api/cluster.html#workerexitedafterdisconnect

```js
if (cluster.isPrimary) {
  for (let i = 0; i < availableParallelism(); i++) cluster.fork();
  cluster.on('exit', (worker) => { if (!worker.exitedAfterDisconnect) cluster.fork(); });
} else {
  http.createServer(handler).listen(8000);
}
```

## Old patterns

| Old pattern | Replacement | Source |
| :-- | :-- | :-- |
| `cluster.isMaster` | `cluster.isPrimary` | https://nodejs.org/docs/latest/api/cluster.html#clusterismaster |
| `cluster.setupMaster()` | `cluster.setupPrimary()` | https://nodejs.org/docs/latest/api/cluster.html#clustersetupmastersettings |
| `worker.terminate(callback)` | `await worker.terminate()` | https://nodejs.org/docs/latest/api/worker_threads.html#workerterminate |
| `os.cpus().length` workers | `os.availableParallelism()` | https://nodejs.org/docs/latest/api/os.html#osavailableparallelism |
