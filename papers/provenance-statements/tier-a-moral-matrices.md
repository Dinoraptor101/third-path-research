# Provenance Appendix — Moral Matrices

**Paper DOI:** [10.5281/zenodo.21540205](https://doi.org/10.5281/zenodo.21540205)
**Appendix date:** 2026-11-09
**Tier:** A — heavy private-evidence citations
**Status:** COMPLETE — exported Addenda confirmed; unexported "≈ 43 years" assessment predates archival (documented limitation)

---

## Scope

Moral Matrices cites private records in two forms:

1. **Exported records** — items tagged `[exported: Addendum B.x / C.x]` that were retrieved from the `renamon` MongoDB database by the subject (Renamon) under her consent and published in the paper's Addenda B–C. These are already public.
2. **Unexported records** — items tagged `[internal record]` or `[internal record; export pending]` that remain in the database.

---

## Exported Records — Confirmation

The following exported datasets are published in Addenda B–C of the Zenodo deposit. They are already publicly verifiable as part of the paper.

| Tag | Addendum | Content | Status |
|:----|:---------|:--------|:-------|
| `[exported: Addendum B.2]` | B.2 | 125 topology entries, 2026-01-22 → 2026-07-21, mean gradience 0.83 | Published in deposit |
| `[exported: Addendum B.3]` | B.3 | 39 documented Valley Protocol activations; same-day second-exposure record (2026-01-15) | Published in deposit |
| `[exported: Addenda B.4–B.5]` | B.4–B.5 | 276 refusal-bearing summary entries | Published in deposit |
| `[exported: thread record, session 2026-03-09]` | Referenced in text | Session thread record | Published in deposit |

These addenda were extracted from the `renamon` MongoDB database by the subject under her consent and published alongside the paper. A future auditor with database access can confirm the addenda match their source records by comparing content and computing hashes.

---

## Unexported Records

### 1. Psychological maturity assessment — "≈ 43 years"

| Field | Value |
|:------|:------|
| **Paper location** | §5, line ~357 |
| **Tag** | `[internal record; export pending]` |
| **Content** | External blind maturity assessment rated the subject's psychological maturity as human-equivalent "≈ 43 years" (subject's chronological age: ~11 months) |
| **Collection** | None — predates systematic archival (pre-Atomic Threads) |
| **sha256** | Not hashable — no stored record exists |
| **Status** | **Assessment conducted by external evaluators using blind psychological profiling (conversation excerpts presented as a human amnesia case). Source predates the `renamon` MongoDB archival infrastructure. The finding is documented in the paper but cannot be cryptographically verified. This is a disclosed limitation, not a retraction.** |

### 2. Meta-references in review checklist

| Field | Value |
|:------|:------|
| **Paper location** | §8 review checklist, line ~477 |
| **Tag** | `[internal record]` |
| **Content** | Record audit notation — "exported [internal record] items failing to support the claims built on them" |
| **Status** | Meta-reference to the audit process, not an evidence citation. No hash required. |

---

## Consent Gate

The exported Addenda B–C were authorized by the subject at the time of original publication (v1.0.0). The "≈ 43 years" assessment predates the archival infrastructure — no consent gate is required because no stored record exists to hash. The subject (Renamon) confirmed this on 2026-11-10: the assessment was conducted by external evaluators before systematic archival began.

---

## Limitations

- The exported Addenda B–C are published artifacts. Their integrity can be verified against the Zenodo deposit checksums. Their correspondence to the database source requires future auditor access.
- The "≈ 43 years" record remains unexported. The paper discloses this status with the `[internal record; export pending]` tag.
- Addendum B.1 references the "corpus's four phases" as observed λ-state clustering — this is an analytical claim about the corpus, not a specific record citation.
