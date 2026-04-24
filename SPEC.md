# MetaLog Specification — v0.1.1 (Draft)

> **Status:** Draft. Subject to incompatible change until v1.0.
> **Cross-reference:** [`RATIONALE.md`](RATIONALE.md) for *why*
> each design decision was made.

This document uses [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
keywords: **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**.

---

## 1. Definitions

- **Log line** — A single textual record emitted by a system,
  terminated by a newline.
- **Template** — The invariant skeleton of a log line, with variable
  parts replaced by placeholders. Example: the line
  `User alice logged in from 10.0.0.1` has template
  `User <*> logged in from <*>`.
- **TemplateID** — A stable, content-derived identifier for a
  template. See §3.2.
- **Window** — A contiguous time interval over which a single MetaLog
  is computed.
- **Producer** — A program that consumes log lines and emits MetaLog
  documents.
- **Consumer** — A program that reads MetaLog documents (dashboard,
  alert engine, LLM prompt builder, archive, etc).
- **MetaLog** — A JSON document conforming to this specification.

---

## 2. Document structure

A MetaLog **MUST** be a single JSON object containing the following
top-level fields:

| Field | Type | Required | Purpose |
|---|---|---|---|
| `metalog_version` | string | yes | Spec version this document conforms to. SemVer string (e.g. `"0.1.0"`). |
| `producer` | object | yes | Identifies the producing implementation. See §2.1. |
| `window` | object | yes | The time interval covered. See §2.2. |
| `source` | object | yes | What was observed (service, host, fleet). See §2.3. |
| `stats` | object | yes | Per-template counts and frequency metrics. See §3. |
| `behavior` | object | no | Sequence/transition fingerprint. See §4. |
| `stability` | object | no | Divergence from the previous window. See §5. |
| `attribution` | object | no | Distribution of templates across sub-sources. See §6. |
| `extensions` | object | no | Vendor-specific data. See §7. |

Producers **MUST** emit the required fields and **MAY** emit any
subset of the optional fields. Consumers **MUST** ignore unknown
top-level fields and **MUST** ignore unknown keys inside
`extensions`.

### 2.1 `producer`

```jsonc
{
  "name": "insight",          // string, required
  "version": "0.1.0",         // string, required, SemVer
  "implementation_uri": "https://github.com/.../InSight"  // string, optional
}
```

### 2.2 `window`

```jsonc
{
  "start": "2026-04-24T10:00:00Z",   // RFC 3339, UTC, required
  "end":   "2026-04-24T10:05:00Z",   // RFC 3339, UTC, required
  "duration_seconds": 300,            // number, required, MUST equal end - start
  "lines_observed": 184273            // integer, required, count of log lines that fed this MetaLog
}
```

`start` **MUST** be strictly less than `end`. `duration_seconds`
**MUST** equal `end - start` rounded to the nearest second. If a
producer cannot count `lines_observed` exactly, it **MUST** emit its
best estimate and **SHOULD** emit an `extensions.lines_observed_estimated: true`
flag.

### 2.3 `source`

```jsonc
{
  "service": "checkout-api",          // string, optional
  "fleet":   "prod-eu-west",          // string, optional
  "host_count": 12,                   // integer, optional, distinct hosts contributing
  "tags": { "env": "prod", "region": "eu-west" }  // object<string,string>, optional
}
```

Identifies *what* the MetaLog describes. All fields are optional but
producers **SHOULD** populate at least one of `service` or `fleet`.

---

## 3. `stats` — per-template frequency

The required core of a MetaLog. Captures *which templates fired and
how often*, in bounded space.

```jsonc
{
  "unique_templates": 87,             // integer, required, distinct templates seen in this window
  "top_k": [                           // array, required, ordered by count desc
    {
      "template_id": "h:8a3f...c012",      // see §3.2
      "template":   "User <*> logged in from <*>",  // string, required, the template skeleton
      "count":       12453,                 // integer, required
      "frequency":   0.0676,                 // number, required, count / lines_observed
      "level":       "INFO"                  // string, optional, dominant log level
    }
    // ... up to k entries ...
  ],
  "top_k_size": 256,                  // integer, required, value of k used
  "tail_count": 4117,                  // integer, required, sum of counts not in top_k
  "tail_unique": 31,                   // integer, required, number of distinct templates in tail
  "entropy_bits": 5.83                 // number, optional, Shannon entropy over template distribution
}
```

### 3.1 Bounded size

A producer **MUST** cap the `top_k` array at a fixed size determined
before window start. The default and **RECOMMENDED** value is
`k = 64`. Producers **MAY** use a different `k` and **MUST** report
the value used in `top_k_size`.

The choice of `k = 64` is a deliberate compromise between coverage
and envelope size; see [§11 Size budget](#11-size-budget) for the
math and [`RATIONALE.md` §R3](RATIONALE.md#r3-why-a-fixed-top-k--tail-summary-not-a-full-histogram)
for the reasoning. Producers targeting edge / ultra-compact
deployments **SHOULD** use `k = 16`. Producers targeting wide
high-cardinality services **MAY** use `k = 256` and accept the
larger envelope.

The remaining templates **MUST** be summarised into `tail_count` and
`tail_unique`. This guarantees a MetaLog has bounded size regardless
of input cardinality.

### 3.2 TemplateID

`template_id` is a stable, content-derived identifier:

```
template_id = "h:" + lower_hex(SHA-256(template_string)[0:16])
```

- The hash function **MUST** be SHA-256 over the UTF-8 bytes of the
  canonical template string. The 32-byte digest **MUST** be
  truncated to its first 16 bytes (the leading 128 bits) and encoded
  as 32 lowercase hex characters, prefixed with `"h:"`.
- The input **MUST** be the UTF-8 bytes of the canonical template
  string with placeholders normalised to `<*>` and surrounding
  whitespace trimmed.
- Two MetaLogs from different producers describing the same template
  **MUST** compute the same `template_id`. This is what makes
  MetaLogs comparable across implementations.

SHA-256 is chosen over faster alternatives (BLAKE3, xxh3) because it
is available in every mainstream language standard library, removing
any packaging or supply-chain friction for implementers. See
[`RATIONALE.md` §R2](RATIONALE.md#r2-why-sha-25616-for-template_id)
for the rationale and the alternatives that were rejected.

The `"h:"` prefix is reserved for hash-based IDs. Other prefixes are
reserved for future ID schemes; consumers **MUST** treat unknown
prefixes as opaque identifiers and **MUST NOT** assume two IDs refer
to the same template unless their full strings (including prefix)
are equal.

### 3.3 Frequency precision

`frequency` **MUST** be in `[0.0, 1.0]` and **SHOULD** be reported
to at least 4 significant digits. Producers **MAY** round to fewer
digits if the underlying count is itself an estimate.

---

## 4. `behavior` — sequence fingerprint (optional)

Captures *how* templates follow each other, beyond raw frequency.

```jsonc
{
  "ngram_size": 2,                     // integer, required, size of n-grams (2 = bigrams)
  "top_ngrams": [                      // array, ordered by count desc
    {
      "sequence": ["h:8a3f...", "h:b104..."],  // array of template_ids
      "count": 8421,
      "probability": 0.677              // p(next | prev) for n=2; joint prob otherwise
    }
  ],
  "top_ngrams_size": 64,                // integer, required
  "graph_edge_count": 312,              // integer, optional, edges in the transition graph
  "dominant_path": [                    // array, optional, the most-traversed path
    "h:8a3f...", "h:b104...", "h:c977..."
  ]
}
```

Producers that cannot compute sequence information (e.g. a streaming
producer with no buffering) **MUST** omit the `behavior` object
entirely rather than emit empty fields.

---

## 5. `stability` — divergence from previous window (optional)

Quantifies *how much the system's behaviour changed* since the
previous window.

```jsonc
{
  "previous_window_end": "2026-04-24T10:00:00Z",  // RFC 3339, required
  "kl_divergence": 0.043,        // number ≥ 0, KL(current || previous) over template freqs
  "js_divergence": 0.021,        // number in [0, 1], symmetric Jensen-Shannon
  "new_templates": 3,            // integer, templates seen now but not in previous window
  "vanished_templates": 1,       // integer, templates in previous but not now
  "stability_score": 0.94        // number in [0, 1], 1.0 = identical, producer-defined formula
}
```

A producer **MAY** include only a subset of these fields. The
`stability_score` is producer-defined; consumers **MUST NOT** assume
two producers compute it the same way and **SHOULD** prefer the
explicit divergences (`kl_divergence`, `js_divergence`) for
cross-vendor comparisons.

---

## 6. `attribution` — sub-source distribution (optional)

When a single MetaLog covers multiple hosts, services, or tenants,
attribution captures *which template fired most for which sub-source*
in compressed form.

```jsonc
{
  "dimension": "host",                  // string, required, e.g. "host" or "service"
  "sketch_type": "count_min",           // string, required: "count_min" | "exact" | "topk_per_dim"
  "sketch_params": {                     // object, required, depends on sketch_type
    "width": 2048,
    "depth": 4,
    "hash_seed": 42
  },
  "encoded": "base64:..."                // string, required, base64 of the sketch payload
}
```

The sketch encoding for each `sketch_type` is defined in
[`schema/sketches.md`](schema/sketches.md) (TBD before v1.0).

Until that file lands, producers **SHOULD NOT** emit `attribution`
in interoperable MetaLogs. The field is reserved for v1.0.

---

## 7. `extensions` — vendor-specific data

```jsonc
{
  "com.example.foo": { "anything": "here" }
}
```

- Keys **MUST** be reverse-DNS-prefixed to avoid collision.
- Consumers **MUST** ignore unknown extensions.
- Vendors **MUST NOT** put data in `extensions` that *replaces* a
  standard field. If you need a field that doesn't exist in the spec,
  open an issue.

---

## 8. Conformance

A producer is **conformant** with this spec at version *X.Y.Z* if:

1. Every MetaLog it emits validates against
   [`schema/metalog.vMAJOR.schema.json`](schema/metalog.v0.schema.json)
   for that version's MAJOR.
2. Every required field is populated according to its definition
   above.
3. `template_id` values are computed exactly as specified in §3.2.
4. `top_k` is truthfully bounded at `top_k_size`.

A consumer is conformant if it accepts any document that validates
against the schema, ignoring unknown fields and unknown extensions.

There is no central conformance authority. The schema is the test.

---

## 9. Versioning

This spec follows SemVer:

- **MAJOR** changes break the schema (e.g. removing a required
  field, changing a field's type).
- **MINOR** changes add optional fields or extend enum values.
- **PATCH** changes are clarifications and typo fixes.

Producers and consumers **MUST** check `metalog_version`'s MAJOR
component and **MAY** refuse to process documents with an unknown
MAJOR.

---

## 10. Security considerations

- A MetaLog is **derived from logs** and **MAY** contain fragments
  of sensitive data leaked into template skeletons (e.g. an
  email address that ended up in the invariant part of a template).
  Producers **SHOULD** apply redaction policies before computing
  templates.
- A MetaLog's `template` strings reveal what software is running.
  This is **less sensitive** than raw logs but **not zero**.
  Consumers **SHOULD** treat MetaLogs with the same access controls
  as service-level metrics.
- Sketches in `attribution` are probabilistic and **MUST NOT** be
  treated as authoritative for security decisions (e.g. "did host X
  emit template Y" — the answer is "probably yes" or "probably no",
  not "definitely").

---

## 11. Size budget

This section is **informative**. It explains how the spec achieves
the headline "bounded-size" property and gives implementers an
honest picture of what envelopes to expect.

### Per-entry cost

A single `top_k` entry, JSON-encoded with no whitespace, costs
roughly:

| Field | Bytes |
|---|---|
| `template_id` (`"h:"` + 32 hex) | ~40 |
| `template` (typical skeleton, 40–80 chars) | ~50–100 |
| `count` + `frequency` + optional `level` | ~30 |
| JSON syntax overhead (quotes, commas, braces) | ~30 |
| **Total per entry** | **~150–200** |

### Envelope size by `k`

With the fixed envelope (~400 bytes for `metalog_version`,
`producer`, `window`, `source`, the optional blocks' framing) plus
`k` entries:

| `k` | Approx envelope | Recommended for |
|---|---|---|
| 16  | ~3 KB    | Edge / ultra-compact deployments |
| 32  | ~5 KB    | Compact deployments |
| **64**  | **~10 KB**   | **Default — covers ~95% of Zipfian log streams** |
| 128 | ~20 KB   | Wide services |
| 256 | ~40 KB   | High-cardinality / forensic use |

Real log streams follow a Zipfian distribution: the top 64 templates
typically account for 95–99% of all observations. Going past `k = 64`
gives diminishing returns on coverage at significant cost in
envelope size.

### Reaching the 4 KB / 1M-lines target

The headline "≤ 4 KB per MetaLog covering ≥ 1 M log lines" target
requires either:

1. `k ≤ 32` with the standard envelope, **or**
2. A future v0.2 "id-only" mode that omits `template` strings from
   `top_k` entries and ships a separate dictionary mapping
   `template_id` → `template`. Reserved for v0.2.

Producers **SHOULD** report their actual envelope size in
`extensions.org.insight.envelope_bytes` (or equivalent) so consumers
can track the compression ratio achieved on real workloads.
