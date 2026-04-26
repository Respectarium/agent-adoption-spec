# Contributing to the Agent-Adoption Specification

We welcome contributions from researchers, implementers, methodologists, and anyone with subject-matter expertise in agent-readiness measurement.

---

## How to contribute

### For non-developers

Open an [Issue](../../issues/new/choose). Pick the template that matches your contribution shape:

- **Propose a check** — suggest a new check or modification to an existing check
- **Report an ambiguity** — flag where the specification is unclear or open to multiple reasonable interpretations
- **Methodology question** — ask about scoring, weights, scope, or the rationale for a specific design decision
- **Other** — anything else

### For developers

Open a Pull Request against `main`. Update `SPEC.md`, `CHANGELOG.md`, and any relevant supporting files. Keep changes scoped — one logical change per PR.

A reasonable PR includes:
- The proposed change to the specification text
- A short rationale (in the PR description) explaining why the change improves the spec
- A note on whether the change is editorial (no behavior change), additive (new optional element), or breaking (changes existing conformance contract)

---

## Decision process

Pull Requests are reviewed by the maintainers (see [`governance/maintainers.md`](governance/maintainers.md)). Acceptance criteria:

1. **Technically sound** — internally consistent with the rest of the specification; doesn't introduce contradictions
2. **Empirically defensible** — proposed check additions or weight changes should reference evidence (the [correlation study](https://github.com/respectarium/agent-adoption-research) is the natural source). Proposals without evidence may be accepted as informational additions but won't be promoted to scored predictors without supporting research
3. **Scoped appropriately** — measures HTTP-level practices an external scanner can verify; does not attempt to specify LLM ranking algorithms or causal-inference claims about visibility outcomes
4. **Backward-compatible where possible** — new checks SHOULD be additive (optional in v1.1) rather than breaking (mandatory new checks in v2.0). Major-version bumps are reserved for genuine breaking changes

---

## Versioning of contributions

Contributions land in different tagged releases depending on their scope:

- **Editorial fixes** (typos, broken links, clarifications without behavior change) → patched in `main`, included in the next patch release (v1.0.x)
- **Additive changes** (new optional checks, new profile bundles, new informational fields) → minor release (v1.1.0)
- **Breaking changes** (renamed checks, removed checks, score-formula changes, gate-criterion changes) → major release (v2.0.0); these typically wait for the next quarterly empirical-research cycle

The current version's release line is documented in [`CHANGELOG.md`](CHANGELOG.md).

---

## What we are NOT looking for

To save reviewer time, the following classes of contributions are unlikely to be accepted:

- **Promotional content** for any specific scanner implementation (the spec is implementation-agnostic; we don't endorse vendors)
- **Branding or naming changes** — the naming convention is locked at the v1.0 release and is not a contribution surface
- **Removal of the conflict-of-interest disclosure** (Respectarium operates one of the conforming implementations; this stays disclosed)
- **Major scope expansions** (e.g., adding outcome-prediction methodologies to the spec) — those belong in the [correlation-research repository](https://github.com/respectarium/agent-adoption-research), not in the specification

---

## License of contributions

By submitting an Issue or Pull Request, you agree your contribution is licensed under [Creative Commons Attribution 4.0 International (CC-BY 4.0)](LICENSE) — the same license as the rest of this repository.

---

## Code of Conduct

Be civil. Critique work, not people. Disagreement on technical content is welcome; ad-hominem and personal attacks are not.

This project follows the spirit of the [Contributor Covenant](https://www.contributor-covenant.org/). Maintainers reserve the right to close issues or block contributors who repeatedly violate this norm after warning.

---

## Getting help

If you have a question that doesn't fit the Issue templates, open a [Discussion](../../discussions). Discussion is a lower-stakes surface than Issues — good for "is this in scope?" or "how should I think about X?" type questions.
