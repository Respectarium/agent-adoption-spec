# Maintainers

This specification is maintained by **Respectarium**.

---

## Maintainer roles

| Role | Responsibility |
|---|---|
| **Methodology lead** | Owns spec content decisions, drafts new versions, reviews methodological PRs |
| **Technical maintainer** | Owns repository administration, merges editorial PRs, manages tags and releases |
| **Empirical-research liaison** | Coordinates with the [correlation-research](https://github.com/respectarium/agent-adoption-research) team to ensure spec evolution is grounded in evidence |

Current named maintainers and contact channels are documented in this repository's GitHub team configuration.

---

## Decision process

Different change classes follow different review paths:

| Change type | Required approval | Notes |
|---|---|---|
| **Editorial fixes** (typos, link fixes, formatting) | 1 maintainer | Direct merge |
| **Methodology clarifications** (no behavior change in conformance) | 1 maintainer | Direct merge with rationale in PR description |
| **Additive changes** (new optional checks, new profiles, new informational fields) | Methodology lead + 1 other maintainer | Lands in next minor version (v1.x.0) |
| **Breaking changes** (renamed checks, score-formula changes, removed checks, new gate criteria) | Methodology lead + technical maintainer + empirical-research liaison | Lands in next major version (v2.0.0); requires published rationale |
| **Errata in a frozen tag** | Recorded in CHANGELOG with pointer to corrected version | Past tag remains immutable |

---

## Tag immutability

Once a specification version is tagged (e.g., `v1.0.0`), the tag is **immutable**. Errors discovered after release are documented in [`CHANGELOG.md`](../CHANGELOG.md) with a pointer to the corrected version in a subsequent tag. The historical tag remains preserved as-is so prior citations remain valid.

This convention is standard for technical specifications — implementers must be able to target a specific version of the spec with confidence that its requirements won't change beneath them.

---

## Empirical-research grounding

Where the specification's behavior is calibrated by empirical findings (e.g., check weights, gate criteria), the rationale is documented in two places:

1. **In this repository** — `SPEC.md` cites the relevant empirical evidence in inline references and in the References section
2. **In the [correlation-research repository](https://github.com/respectarium/agent-adoption-research)** — the underlying data and statistical analyses that informed the calibration

Major-version bumps (v2.0, v3.0, etc.) are tied to the quarterly correlation-research cycle. New evidence that materially changes which checks predict outcomes triggers a v(N+1).0 conversation with the empirical-research liaison.

---

## Contact

- **Issues + Discussions:** open via GitHub on this repository — these are the primary contact channels for questions, proposals, replication reports, and ambiguity reports
- **Private inquiries:** open a [Discussion](../../discussions) marked as private; or reach out via the contact channels published on respectarium.com
- **Security concerns** about the specification's design (rare — this is a methodology spec, not an active service): open a [private security advisory](../../security/advisories) via GitHub
