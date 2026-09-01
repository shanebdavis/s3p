# NoSuchKey error crashes s3p instead of being handled as a warning

## Bug Report / Feature Request

When operating on a bucket whose contents change during a `cp` or `sync` operation (e.g. objects are deleted by another process), s3p crashes with an unhandled `NoSuchKey` error instead of logging a warning and continuing.

### Expected behavior

When a key that was discovered during listing is deleted before the copy attempt, s3p should:
1. Log a **warning** (visible by default, not only in `--verbose` mode)
2. Track the skipped files in the final stats (`keyDoesNotExistFiles`, `keyDoesNotExistBytes`)
3. **Continue** processing the remaining items

### Actual behavior

s3p crashes with a fatal error propagating through `eachRecursive`:

```
eachRecursive:
  startAfter: ...
  stopAt: ...
  usePrefixBisect: false
  error: Error:
    class: class Error
    stack:
      NoSuchKey: The specified key does not exist.
          at ...
```

The entire operation aborts.

### Root cause

The existing `NoSuchKey` catch in `_copyWrapper` (added in v3.5.1) has two issues:

1. **Fragile error detection**: It uses `error.stack` regex matching (`/NoSuchKey: The specified key does not exist/`), which doesn't reliably match all SDK error formats. In particular:
   - AWS SDK v3's `headObject` (used in `largeCopy` for files ≥ 100MB) throws with `error.name === "NotFound"` rather than `"NoSuchKey"`
   - Stack trace formats vary between SDK versions

2. **`copyingBytesInFlight` not decremented on error**: When a copy fails with `NoSuchKey`, the bytes-in-flight stat is never decremented (only the `.then` path decrements it), which can affect progress reporting and throttling.

3. **Warning only shown in verbose mode**: The skip message required `--verbose` to be visible, so users had no indication that files were being silently skipped.

### Environment

- s3p version: 3.7.2 (also affects earlier versions back to 3.5.1)
- AWS SDK: v3 (@aws-sdk/client-s3 ^3.598.0)
- Node.js: any supported version

### Steps to reproduce

1. Start a `cp` or `sync` operation on a bucket with active writes/deletes
2. While s3p is running, delete an object that s3p has already listed but not yet copied
3. Observe the crash

### Fix

The catch block should:
- Check `error.name` (`"NoSuchKey"` or `"NotFound"`) for reliable detection across SDK operations
- Fall back to regex on `error.message` for older SDK versions
- Decrement `copyingBytesInFlight` on error (same as on success)
- Log a warning by default (suppressed only by `--quiet`)
