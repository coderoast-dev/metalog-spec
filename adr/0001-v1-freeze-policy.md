# ADR 0001: MetaLog v1 Freeze Policy

## Status

Accepted.

## Context

MetaLog is still in the v0 draft line. v0.2.0 added template deduplication, formal behavior fields, composition, sessions, and `MetaLogDiff`. InSight is the reference implementation path, but other consumers need a clear rule for when the format stops breaking.

## Decision

Keep v0.x explicitly unstable. Freeze v1.0 only after:

- InSight emits and consumes the full v0.2 envelope in integration tests.
- `MetaLogDiff` is implemented against consecutive and arbitrary windows.
- Composition is validated across shards and time windows.
- CodeRoastServer and CodeRoastWeb contract docs identify which MetaLog fields they expose or display.
- The compatibility matrix names the InSight package version that implements the freeze candidate.

After v1.0, breaking schema changes require a new major version and a migration note in `CHANGELOG.md`.

## Consequences

- v0.x schemas may still change incompatibly.
- Consumers must pin `metalog_version` and tolerate unknown fields.
- Release notes must call out draft-breaking changes clearly until v1.0.