# Plan: HTTP Socket Pool Sizing

Status: **proposed** (not started)
Origin: runtime warning —

```
@smithy/node-http-handler:WARN - socket usage at capacity=50 and 450 additional requests are enqueued.
See https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/node-configuring-maxsockets.html
or increase socketAcquisitionWarningTimeout=(millis) in the NodeHttpHandler config.
```

## Problem

The AWS SDK v3's `NodeHttpHandler` defaults to an https agent with `maxSockets: 50`. s3p's worker pools are sized far larger — `copyConcurrency = 500`, `deleteConcurrency = 500`, `listConcurrency = 100` (`S3Comprehensions.caf:24-27`, `S3P.caf:244`) — and `S3.caf` never configures a `requestHandler`. The numbers in the warning map exactly: 50 requests on the wire, the other 450 copy workers queued inside the HTTP handler.

Consequences:

- **Concurrency flags are an illusion.** Effective concurrency is capped at 50 regardless of `--copy-concurrency`; 90% of the pool just relocates the queue from s3p's `PromiseWorkerPool` into the SDK.
- **Distorted timings.** Socket-queue wait counts toward request duration, so the `S3.list-slow` check (`S3.caf:84`, duration > 60s) can fire on socket-starved requests rather than slow S3 responses. Any timeout tuning is similarly polluted.
- **The warning's own suggestion is a trap.** Raising `socketAcquisitionWarningTimeout` silences the message without touching the bottleneck.

## Fix

Configure the request handler in the `S3` constructor (`S3.caf:32`):

```js
import { NodeHttpHandler } from '@smithy/node-http-handler'  // already a transitive dep of client-s3
import { Agent } from 'https'

new S3Client({
  region,
  requestHandler: new NodeHttpHandler({
    httpsAgent: new Agent({ keepAlive: true, maxSockets }),
  }),
})
```

- `maxSockets` becomes an `S3` constructor option, threaded from the callers:
  - copy/sync paths: default `≈ copyConcurrency + listConcurrency` (list and copy run simultaneously against the same client).
  - delete path: default from `deleteConcurrency`.
- Expose a CLI flag (`--max-sockets`) for override; document that it normally follows the concurrency flags automatically.

## Guardrails

- **File descriptors:** 600 sockets exceeds default `ulimit -n` on macOS (256) and stock Linux (1024). Clamp with a warning, or document raising the limit — decide during implementation.
- **S3 rate limits become reachable:** at a true 500-way COPY against one prefix, S3's ~3,500 mutating-requests/sec/prefix limit is hittable, surfacing `503 SlowDown`. The SDK's adaptive retry handles it, but note in docs that some users' current stability is an accident of the 50-socket throttle — raising it may surface retries they've never seen.

## Testing

- Unit: constructor passes `maxSockets` through to the agent; defaults derive from concurrency options.
- Integration (minio): run a copy with `copyConcurrency > 50` and assert no `@smithy/node-http-handler` capacity warning is emitted; sanity-check throughput scales past the 50-socket ceiling.

## Interaction with other plans

Independent of, and shippable before, `cross-region-cross-account-copy.md`. If that plan lands, note that its stream mode holds sockets for the full transfer duration on both sides, so the `fromS3` and `toS3` clients need independently sized pools.

## Effort estimate

Small — a few hours including tests. The only real decision is default sizing vs. ulimit clamping behavior.
