# Changelog

All notable changes to the MetaLog specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
The spec follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> **Pre-1.0 notice:** during the 0.x line, MINOR version bumps may
> still introduce incompatible schema changes. After 1.0, semver
> applies strictly.

---

## [0.1.0] — 2026-04-24

### Added
- First public draft.
- Top-level envelope: `metalog_version`, `producer`, `window`,
  `source`, `stats`.
- Optional sections: `behavior`, `stability`, `attribution`,
  `extensions`.
- `template_id` definition: `"h:" + lower_hex(BLAKE3-128(canonical_template))`.
- Bounded `top_k` + tail-summary design (default `k = 256`).
- KL / JS divergence fields in `stability` for cross-vendor
  comparability.
- JSON Schema at `schema/metalog.v0.schema.json`.
- Worked example at `schema/metalog.v0.example.json`.
- `RATIONALE.md` documenting why each design decision was made and
  what was rejected.

### Status
- `attribution` is reserved but its sketch encoding is not yet
  pinned. Producers should not emit `attribution` in interoperable
  MetaLogs until v1.0.
