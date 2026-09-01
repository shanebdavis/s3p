# "socket hang up" (TimeoutError) aborts large cp/sync operations

## Bug Report / Feature Request

When running a large `cp` or `sync` at high concurrency, s3p dies after many minutes with a cascade of
`TimeoutError: socket hang up` errors, aborting the whole run and discarding hundreds of thousands to millions of items
of progress.

### Expected behavior

Transient connection-level failures (the connection is reset mid-request) are momentary and safe to retry. s3p should:

1. Retry the failed S3 operation (list, copy, or delete) a few times with backoff
2. Log a **warning** per retry (visible by default, not only in `--verbose` mode)
3. **Continue** the operation, so one dropped socket does not abort the whole run

### Actual behavior

A single `socket hang up` propagates up through `eachRecursive` and aborts everything:

```
eachRecursive:
  startAfter: ...
  stopAt: ...
  usePrefixBisect: false
  error: Error:
    class: class Error
    stack:
      TimeoutError: socket hang up
          at TLSSocket.socketOnEnd (node:_http_client:792:25)
          at TLSSocket.emit (node:events:526:24)
          at endReadableNT (node:internal/streams/readable:1772:12)
          at process.processTicksAndRejections (node:internal/process/task_queues:90:21)
...
Hangup
```

The entire operation aborts, no matter how far it had progressed.

### Root cause

Two layers were missing:

1. **The S3 client configured no retry policy or request timeouts.** It silently ran on the AWS SDK's implicit
   `standard` / 3-attempt default and a socket with no idle/connection timeout, so a stuck connection could hang and a
   burst of drops could exhaust the SDK's few attempts.

2. **The AWS SDK v3 retry classifier does not reliably treat connection-reset errors as retryable**
   (see [aws-sdk-js-v3 #4666](https://github.com/aws/aws-sdk-js-v3/issues/4666)). A raw `TimeoutError` /
   `socket hang up` (and the underlying socket codes `ECONNRESET`, `ETIMEDOUT`, `EPIPE`, `ENOTFOUND`, ...) lacks the
   markers the SDK's classifier looks for, so the SDK gives up immediately and the error propagates. Because these
   errors surface on the parallel `listObjectsV2` calls **and** the `copyObject` / `uploadPartCopy` calls that
   `eachRecursive` awaits, a single one aborts the whole sync.

### Environment

- s3p version: 3.7.2 (all prior versions affected)
- AWS SDK: v3 (@aws-sdk/client-s3 ^3.598.0)
- Node.js: any supported version
- Most visible at high concurrency (e.g. `--copy-concurrency 200 --list-concurrency 50`)

### Steps to reproduce

1. Run a `sync` or `cp` over a large bucket (hundreds of thousands of objects) at high concurrency
2. Let it run for several minutes so the copy pool saturates
3. Observe the `TimeoutError: socket hang up` cascade and the abort

### Fix

**SDK-level hardening** (helps reach/stability; handles throttling and 5xx):

- Adaptive retry mode with `maxAttempts: 5` (was the SDK's implicit `standard` / 3). Adaptive mode adds client-side rate
  limiting on `503 SlowDown`.
- `connectionTimeout` (6s) and `requestTimeout` (120s, with `throwOnRequestTimeout`) on the `NodeHttpHandler`, so a
  stuck socket becomes a fast retryable failure instead of an indefinite hang.
- Thread `maxAttempts` / `connectionTimeout` / `requestTimeout` through every client construction site (copy source +
  target, delete, both list clients), all overridable. Expose a `--max-attempts` CLI flag.

**Application-level retry** (the actual fix — the SDK does not retry connection-reset errors):

- `_isTransientError` classifies the connection-reset family the SDK misses: `TimeoutError` / `socket hang up` and the
  raw socket codes `ECONNRESET`, `ETIMEDOUT`, `ECONNREFUSED`, `EPIPE`, `ENOTFOUND`, `EAI_AGAIN`, `ECONNABORTED`,
  `ERR_SOCKET_CONNECTION_TIMEOUT`.
- `_withTransientRetry` retries with exponential backoff (250ms base, 5s cap, full jitter) up to `maxAttempts`.
  Non-transient errors (`NoSuchKey`, `AccessDenied`, ...) rethrow immediately.
- Wrap the `list` (`listObjectsV2`), `copy` (server-side `copyObject`), `largeCopy` (per-part `uploadPartCopy`) and
  `delete` (`deleteObject`) calls. All are idempotent or side-effect-free, so retrying is safe.
- Emit one compact log line per retry:
  ```
  s3p: transient-retry 1/5 in 158ms - copy <bucket>/<key> - TimeoutError: socket hang up
  ```

### Note on high concurrency

At very high concurrency the retries can fire in large correlated bursts, which is a sign of **client-side socket
saturation** (the machine opening more concurrent TLS connections than its network stack sustains). Retries are the
safety net; if you see storms of `transient-retry` lines, reduce `--copy-concurrency` / `--list-concurrency` (or run s3p
on an EC2 instance in-region). Retries are not a substitute for right-sizing concurrency.
