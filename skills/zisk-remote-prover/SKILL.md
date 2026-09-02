---
name: zisk-remote-prover
description: Use when submitting ZisK work to a coordinator or worker fleet, using RemoteClient, ZiskStream, streamed stdin or hints, diagnosing remote job lifecycle failures, or deploying/operating ZisK distributed proving.
license: MIT
metadata:
  version: "1.0.0"
  domain: zkvm
  triggers: ZisK remote, RemoteClient, ZiskStream, coordinator, zisk-coordinator, worker, zisk-worker, distributed proving, streamed hints, QUIC, gRPC, job handle, proving cluster
  role: specialist
  scope: distributed-proving
  output-format: operational-report
  related-skills: zisk-developer, zisk-build, zisk-soundness, devops-engineer
---

# ZisK Remote Prover

Use this skill to make a remote proof request a traceable job, not a best-effort HTTP
call. The coordinator may execute a valid job for the wrong ELF, incomplete stream,
or wrong setup unless the client records and checks those boundaries.

## Scope and Routing

Own remote SDK usage, coordinator/worker lifecycle, `ZiskStream`, input/hint transport,
job observation/cancellation, and deployment operations. Use `zisk-build` for local
ELF/setup identity, `zisk-developer` for guest/I/O/hints semantics, and
`zisk-soundness` when a returned proof will authorize anything.

## Remote Job Contract

Record this before submitting a production job:

```text
coordinator URL and authenticated transport/ingress policy
ZisK revision; client and worker binary revisions
ELF digest/program hash_id; hash mode; setup/key identity
operation: upload | setup | execute | prove | wrap | aggregate
executor and hint mode; buffered input digest or stream URI/transport
timeout, cancellation owner, expected public output/proof consumer
job ID and terminal result/artifact digest
```

The uploaded program identity is the coordinator `hash_id`; current remote upload
checks the coordinator's returned identity against the locally computed program ID.
Preserve that check and do not substitute a human program name for the hash.

## Buffered Input versus Streams

Choose `ZiskStdin` for a complete, bounded input that can be hashed and retained.
Choose `ZiskStream` only when incremental or write-after-start delivery is needed.
The two are different `InputSource` variants, not interchangeable transport details.

`ZiskStream` methods have exact wire semantics:

| Writer | Wire form | Guest pairing |
| --- | --- | --- |
| `write(&T)` | one bincode record, length-prefixed and 8-byte padded | `ziskos::read::<T>()` |
| `write_slice(bytes)` | one raw framed record, length-prefixed and padded | `read_slice` / pinned raw-input API |
| `write_bytes(bytes)` | unframed bytes | only a guest that explicitly parses that raw layout |

For a framed record, the header is an eight-byte little-endian length followed by the
payload and zero padding to an eight-byte boundary. Never send bare application bytes
to a guest calling a framed read.

Transports currently include Unix sockets, QUIC, and coordinator gRPC push. Capture the
URI given to the job; do not expose a generated Unix path or an unauthenticated network
endpoint as a stable public API without a deployment review.

## Stream Lifecycle

```text
construct -> write buffered records -> submit/start -> flush while live -> finish/await -> terminal job
```

- `flush()` may block until the peer/sender becomes live. Bound it with the job's
  timeout/cancellation plan; do not call it from an unbounded request handler.
- A gRPC stream receives its sender only after job submission. A Unix/QUIC stream must
  be listening before the executor connects.
- Awaiting a `JobHandle` closes associated streams automatically. On error paths where
  no handle is awaited, call `finish()` so the peer sees EOF and resources are released.
- A stream can be reused only after the previous lifecycle is closed; `finish()` makes
  later `flush()` wait for a new start rather than leaking data into a completed job.
- Test early EOF, late data, frame truncation, duplicate records, and cancellation.

Hints have the same ordering and integrity requirements as normal input. Streaming is
delivery, not authentication: the guest must verify every security-relevant hinted
result against its theorem.

## Job Lifecycle and Recovery

Follow the remote flow deliberately:

```text
load exact ELF -> upload/confirm hash_id -> setup for its mode -> submit job
  -> observe job events/status -> retrieve terminal result -> verify proof/claim
```

Do not assume a client retry is safe. First discover whether the original job exists,
its terminal state, and whether a streamed input was fully closed. A retry can create a
second expensive proof or a different execution if its source stream differs.

For each phase distinguish retryable transport failure from a rejected job, worker loss,
proof failure, timeout, or cancellation. Preserve job ID, coordinator logs/events, worker
identity/capacity, program hash, input/hint digest or stream trace, and setup identity.
Use cancellation for jobs no longer wanted; then wait for a terminal status before
releasing streams or assuming capacity is free.

Remote execute is not remote prove. A successful execute response proves only the
selected execution path ran; apply the requested `prove`/`wrap`/`aggregate` flow and
then the real consumer verification for any proof-bearing result.

## Deployment Baseline

The upstream deploy tree supplies Docker Compose, host scripts, Ansible, and a worker
Helm skeleton. Treat them as revision-specific starting points, not a production
security policy.

- Start from one coordinator and explicit workers; the upstream Kubernetes material
  models one coordinator per cluster and workers that dial outward, so workers do not
  need an inbound Service merely to receive work.
- Persist and monitor worker cache/proving-key directories. A cold worker can look
  healthy while spending its capacity rebuilding setup.
- Pin the coordinator and worker binary/bundle versions together. Roll upgrades with
  queue drain, job compatibility checks, and a rollback plan.
- Expose metrics deliberately. The upstream Docker development stack enables Grafana
  anonymous admin access; disable that outside a local development environment.
- Put coordinator access behind explicitly configured authentication, TLS/ingress,
  network policy, quotas, and artifact retention. Do not infer any of those controls
  from the default local URL.
- Scale on queue depth and proven capacity where available, not CPU alone. GPU/MPI,
  memory, disk cache, and worker configuration are part of capacity; measure them.

## Incident Checklist

| Symptom | First evidence to collect |
| --- | --- |
| Job never starts | coordinator reachability, job ID/status, worker registration/capacity, queue depth |
| Stream blocks | stream URI/transport, submission happened, `flush`/`finish` trace, timeout/cancel state |
| Wrong program/setup | local ELF digest, returned `hash_id`, hash mode, worker cache/setup identity |
| Worker repeatedly fails | binary/bundle parity, executor/GPU/MPI mode, memory/disk, worker logs |
| Proof returns but consumer rejects | proof family/public claim and pinned verifier policy; hand off to `zisk-soundness` |

## Upstream Drift Gate

Before changing a deployment or SDK integration:

```bash
git fetch origin
git diff <last-verified-zisk-commit>..origin/main -- \
  sdk/src/remote sdk/src/input_stream.rs sdk/src/input_source.rs sdk/src/job_handle.rs \
  distributed/crates distributed/deploy cli/src/commands/user/remote
```

Re-test stream framing/lifecycle and job handling if the diff changes input transports,
DTOs, job states/events, retries, timeout/cancellation, cache keys, worker setup, ports,
or deployment templates. Report previous/current revisions and the rollout outcome.

## Validation

Run a tiny known guest through the exact deployment:

```text
buffered execute -> streamed execute -> streamed error/EOF -> remote prove
-> independent proof/claim verification -> cancellation and retry exercise
```

For every successful run retain the contract record, input/stream evidence, job ID,
terminal state, output/proof digest, coordinator/worker revisions, and the command or
SDK configuration actually used.

## Must / Must Not

**Must:** bind work to the returned program hash; close streams; bound waiting and
retries; distinguish execution from proof; record setup/worker versions; validate both
happy and EOF/cancellation paths.

**Must not:** frame data twice or omit framing; reuse a live stream across jobs; assume
default local transport is secure; discard a job ID after a timeout; call a remote
execution result a proof; expose development Grafana/admin defaults in production.
