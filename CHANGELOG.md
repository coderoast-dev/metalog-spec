# Changelog

All notable changes to the MetaLog specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
The spec follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> **Pre-1.0 notice:** during the 0.x line, MINOR version bumps may
> still introduce incompatible schema changes. After 1.0, semver
> applies strictly.

---

## [0.1.1] — 2026-04-24

### Changed
- **`template_id` hash function: BLAKE3-128 → SHA-256 truncated to 128 bits.**
  First-contact with the C++ ecosystem (Conan Center) revealed BLAKE3 is
  not universally packaged, while SHA-256 is in every mainstream language
  standard library (Python `hashlib`, Go `crypto/sha256`, Rust `sha2`,
  JS Web Crypto, openssl for C++). Implementer friction outweighs the
  perf win at template-creation rate (templates are hashed once per
  unique template, not per log line). See updated
  [RATIONALE §R2](RATIONALE.md#r2-why-sha-25616-for-template_id).
- **Default `top_k_size`: 256 → 64.** The size budget math (now
  documented in [SPEC §11](SPEC.md#11-size-budget)) made it explicit
  that 256 entries blow a reasonable envelope by ~10×. 64 covers
  ≥ 95% of Zipfian log streams in ~10 KB. Producers may still pick
  16 (edge), 32 (compact), or 256 (high-cardinality) explicitly.

### Added
- **§11 Size budget** (informative): per-entry cost table, envelope
  size by `k`, and an honest discussion of the 4 KB / 1M-lines target
  (reachable at `k ≤ 32`, or with a future v0.2 "id-only" mode).
- Updated example: `metalog_version: 0.1.1`, `top_k_size: 64`,
  template_ids are real SHA-256[:16] hashes of the example templates
  (verifiable by anyone with `hashlib`).

### Status
- `attribution` remains reserved for v1.0.
- v0.1.x is still draft; MINOR bumps may break compatibility until v1.0.

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
