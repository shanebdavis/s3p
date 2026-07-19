# Plan: Cross-Region & Cross-Account Copy Support

Status: **Phase 0 implemented** (branch `feature/compare-cross-env`, 2026-07); Phases 1–5 not started
Origin: customer report — "s3p couldn't reliably handle the cross-region, cross-account S3 transfer with the separate credentials and regional configuration required for our setup."

## Phase 0 (DONE) — Cross-environment compare & pretend sync

Shipped ahead of the copy work so discrepancies can be *detected* across environments today:

- Per-side clients for listing: `fromS3`/`toS3` in `S3Comprehensions.each`; compare-listing now runs against the target's environment (and reuses one shared client when the sides match — previously `_compareList` constructed a fresh client per recursion step).
- Config via SDK-native profiles: `S3` class accepts `profile` (S3Client config option, SDK ≥ 3.598 — no `@aws-sdk/credential-providers` dependency needed) and `credentials`. CLI: `--profile`, `--endpoint`, `--from-profile`/`--to-profile`, `--from-region`/`--to-region`, `--from-endpoint`/`--to-endpoint`.
- Real cross-environment copying (any `to-profile`/`to-endpoint`/`to-credentials` without `--dryrun`) throws a clear not-yet-implemented error — Phase 3 removes that guard.
- Fixed pre-existing bug: `normalizeOptions` dropped the `returning`/`into` aliases, so `compare` returned only listing stats instead of its counts/bytes discrepancy summary.
- Test infra: second minio container (different port *and* credentials) in docker-compose; cross-env integration tests incl. real profile-file resolution. Run with `S3_ENDPOINT=http://localhost:9000 S3_ENDPOINT_B=http://localhost:9010 AWS_ACCESS_KEY_ID=testAccessKey AWS_SECRET_ACCESS_KEY=testSecretKey AWS_REGION=us-east-1 npm test`.

## Problem

The report is architecturally accurate. Three independent design decisions each block the scenario:

1. **One `S3Client` for everything.** `S3.caf` constructs a single client (one region from `--region`/`AWS_REGION`, one optional `S3_ENDPOINT`), and `S3P._copyWrapper` uses that one instance to list the source, head-check the destination, and copy. But `ListObjectsV2` must hit the *source* bucket's regional endpoint while `CopyObject` must be sent to the *destination* bucket's region — a single-region client gets `301 PermanentRedirect` on whichever side doesn't match. `followRegionRedirects` is not set (and would cost an extra round trip per call at s3p volumes anyway).
2. **No way to express two credential sets.** Credentials are implicit (default provider chain only — no `--profile`, no per-side options). Server-side `CopyObject` requires a *single* principal with read on source and write on destination; "separate credentials per account" cannot be expressed at all.
3. **Large objects take a different path.** `largeCopy` shells out to `aws s3 cp`, which resolves credentials/region independently from the Node SDK with no flags passed through. Small objects succeed while large ones fail (or copy under a different identity) — the likely source of "couldn't *reliably* handle."

Secondary: cross-account copies without `--acl bucket-owner-full-control` leave objects owned by the source account (unless the destination bucket has Object Ownership "bucket owner enforced").

## Current-code facts (verified 2026-07)

- `S3.caf:32-39` — constructor builds one `S3Client`; region from option or `process.env.AWS_REGION`; endpoint from `S3_ENDPOINT`.
- `S3P.caf:389` (and `S3P.caf:256`, `S3Comprehensions.caf:270`) — single `S3` instance created and shared for the whole operation.
- `S3.caf:181` — small-object copy is server-side `CopyObjectCommand`.
- `S3.caf:188-216` — `largeCopy` shells out to `aws s3 cp`; no `--profile`/`--region` passed.
- `LibMisc.caf:87-89` — `parseS3Url` already parses `toRegion` from destination URLs, but **nothing consumes it**.
- `S3.caf:12-23` — `ACL` and other common copy options are already passed through to the SDK.
- `docker-compose.yml` — minio-based integration test infra exists.

### Pre-existing bug to fix first

`S3.caf:152` passes `client: @s3` to `lib-storage`'s `Upload`, but `@s3` is the getter's plain command-wrapper object, not the `S3Client`. `Upload` calls `client.send(...)`, so this should be `@awsSdkS3`. Affects the existing local→S3 large-upload path; fix and add a test before building on this code.

## Correct SDK v3 patterns (reference)

**Cross-region, one credential set** — server-side copy works; send `CopyObjectCommand` to a client configured for the *destination* region. `CopySource` may reference a bucket in any region. Listing needs a *source*-region client, so two clients are required even in the single-credential case.

**Objects > 5 GB** — `CopyObject` hard-fails at 5 GB. Use server-side multipart: `CreateMultipartUpload` → parallel `UploadPartCopyCommand` (byte-range per part, zero data through the machine) → `CompleteMultipartUpload`, all against the destination-region client. This replaces the `aws s3 cp` shell-out.

**Separate credentials per side** — server-side copy is impossible (one request = one signer). Stream through the machine:

```js
const { Body, ContentType, CacheControl, Metadata } = await fromClient.send(
  new GetObjectCommand({ Bucket: srcBucket, Key: srcKey }))
await new Upload({
  client: toClient,
  params: { Bucket: destBucket, Key: destKey, Body, ContentType, CacheControl, Metadata },
  queueSize: 4, partSize: 16 * 1024 * 1024,  // bounds memory per transfer
}).done()
```

**Region discovery** — `HeadBucketCommand` returns `BucketRegion` in current SDK versions; on 301/403 the region is still in the `x-amz-bucket-region` response header. One probe per side at startup.

**Profile credentials** — `fromIni({ profile })` from `@aws-sdk/credential-providers`.

## Plan

### Phase 1 — Per-side clients (plumbing)

- Extend the `S3` class constructor to accept `{region, endpoint, credentials}` explicitly.
- `S3P._copyWrapper` constructs `fromS3` and `toS3`:
  - `fromS3`: listing (`S3Comprehensions`), `GetObject` reads.
  - `toS3`: `CopyObject`, `headObject`/`shouldSyncObjects` comparisons, uploads.
- When no per-side options are given, both names point at one shared instance — existing behavior and test suite unchanged.
- Region resolution order per side: explicit flag → region parsed from the s3/https URL (wire up the already-parsed `toRegion`) → `HeadBucket` probe → `AWS_REGION`. Resolving regions correctly here fixes the 301-redirect failure mode for all users, not just cross-account ones.

### Phase 2 — CLI surface

- New options: `--from-profile` / `--to-profile`, `--from-region` / `--to-region`, `--from-endpoint` / `--to-endpoint`.
- Existing `--region` / `S3_ENDPOINT` remain as defaults applied to both sides.
- Add `@aws-sdk/credential-providers` dependency for `fromIni`.
- Mirror all of this in the programmatic API options.

### Phase 3 — Copy strategy selection

Branch in `S3.copy` on whether the two sides share credentials:

- **Shared credentials** (default): server-side `CopyObject` via `toS3` (today's path, now region-correct). For objects ≥ `largeCopyThreshold`: SDK multipart `UploadPartCopy` fanned out over the existing `PromiseWorkerPool`. Delete the `aws s3 cp` shell-out — removes the CLI dependency, the split credential resolution, and the small/large reliability divergence.
- **Distinct credentials**: streamed `GetObject(fromS3)` → `Upload(toS3)`. s3p already streams S3→disk and disk→S3 in this same function; this composes the two halves. Metadata (`ContentType`, `CacheControl`, `Metadata`, etc.) must be carried explicitly — server-side copy preserves it automatically, the stream path does not.
- Detection: explicit and dumb — any `--to-profile`/to-credential option set ⇒ stream mode. Override with `--copy-mode server|stream` for odd cases (e.g. same creds but VPC/egress reasons to force one mode).

### Phase 4 — Guardrails & semantics

- Stream mode moves every byte through the machine: give it its own concurrency cap (separate, much lower default than `copyConcurrency`, which is sized for cheap server-side calls) to bound memory and NAT bandwidth.
- Cross-account server-side copies: warn (or default) `--acl bucket-owner-full-control` when the two sides' identities differ; document the Object Ownership interaction.
- Progress/stats: stream mode can report real byte progress per part; keep stats shape identical across modes.

### Phase 5 — Tests & docs

- Extend the minio setup to two containers with different keys — simulates two accounts *and* two endpoints end-to-end (list from A, stream to B, verify metadata preservation, verify large-object multipart both modes).
- Unit tests: region resolution order, strategy selection, `toRegion` URL wiring.
- README: new options, cross-account recipe, ownership/ACL note, migration note for anyone who relied on the `aws` CLI shell-out.

## Explicitly out of scope

- The parallel listing engine (`S3Comprehensions` splitting logic) — untouched; this is all in the thin client layer around it.
- Assumed-role orchestration (users can point a profile at a role via standard AWS config).
- Anything beyond two sides (no multi-destination fan-out).

## Open questions

- `largeCopyThreshold` default once shell-out is gone — keep as multipart cutoff, or align to the 5 GB `CopyObject` hard limit with multipart only above it? (Multipart below 5 GB can still be faster via parallel `UploadPartCopy`.)
- Should `--to-region` alone (same creds) also be accepted as a hint even though HeadBucket auto-detect makes it optional? (Cheap yes.)
- Stream-mode default concurrency — needs empirical tuning; start conservative (e.g. 8–16) and expose the flag.

## Effort estimate

Phases 1–2 are careful threading, roughly a day or two. Phase 3 is the substantive new code but leans on `lib-storage` for the hard parts. Phases 4–5 round it out. Ballpark: a focused week including tests.
