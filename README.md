# framecast-api-contracts

Protobuf contracts for the Framecast platform. The schema is the only source of
truth kept here; per-language bindings are generated and hosted by the
[Buf Schema Registry](https://buf.build) (BSR), not committed to this repo.

- Module: `buf.build/framecast/framecast-api-contracts`

## Layout

- `proto/video_processing/v1/` — the worker↔MPS protocol (`JobStream`,
  `WorkerMessage`/`MpsMessage` envelopes, job lifecycle, upload finalization).
- `proto/identity/v1/` — identity service events published to Kafka
  (`AccountVerificationRequested`, `UserRegistered`).

## Consuming

Consumers pull generated SDKs from the BSR with their native package manager —
no code generation in consumer repos. Each consumer authenticates once with a
BSR token (membership in the `framecast` org) and configures its package
manager (e.g. `NuGet.config` for .NET, `.netrc` + `go get buf.build/gen/go/...`
for Go, the BSR Cargo registry for Rust). The exact package name and version
syntax per language are on the module's BSR page.

## Publishing

`buf push` publishes the schema; the BSR regenerates SDKs. CI does this
automatically (`.github/workflows/buf.yml`, authenticated with the `BUF_TOKEN`
secret): pushes publish to the BSR, pull requests run lint/format/breaking. To
push by hand:

```sh
buf registry login buf.build
buf push
```

## Compatibility rules

Contracts are frozen additive-only:

- New fields and new messages may be added; existing fields are never renamed,
  renumbered, retyped, or removed. Field numbers are never reused.
- Enum variants may be added; existing variants keep their numbers. Consumers
  must handle unknown variants explicitly (the worker rejects unsupported
  `SourceProvider` values rather than guessing).
- Secret-bearing fields are documented as such in the proto (e.g.
  `GeneratedKey.key_hex`) and must never be logged or persisted by consumers.

## Checks

Run from the repo root:

```sh
buf lint
buf breaking --against 'buf.build/framecast/framecast-api-contracts'
```

`buf breaking` compares the working tree against the schema published on the
BSR (the baseline consumers build from) using the `FILE` rule set configured in
`buf.yaml`. Run it before every proto-changing commit; CI runs the same checks
on pull requests (`.github/workflows/buf.yml`).

## Ownership

| Contract | Owning implementation | Consumers |
|---|---|---|
| `JobStream` protocol (`worker.proto`) | media-processing-service (server) | encoding-worker (client) |
| `Schema.descriptor_json` payload | encoding-worker `worker-descriptor` crate | media-processing-service, builder UI |
| Compiled wire manifest (`Job.manifest_json`) | media-processing-service `compile` package | encoding-worker core |
| Kafka result message (`mps.jobs.result`) | media-processing-service `kafka` adapter | downstream services |
| `identity/v1` events | framecast-identity-service | notification, profile/metadata, analytics |

Each owning implementation carries a golden/pinned test that freezes the
serialized shape; an intentional contract change regenerates the fixture in the
same review.
