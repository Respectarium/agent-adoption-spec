# Agent-Adoption Specification

The **Agent-Adoption Specification** defines an open methodology for measuring how accessible a website is to AI agents and automated HTTP clients — a category that spans modern crawler-class clients used by LLM training, retrieval-augmented systems, and autonomous-agent architectures.

This repository hosts the canonical specification text, version history, and contribution process. The companion **live human-readable page** is published at [`respectarium.com/spec/agent-adoption/v1`](https://respectarium.com/spec/agent-adoption/v1).

**Current version:** v1.0.0 (2026-04-26)

---

## What this specification does

- Defines a list of **measurable HTTP-level practices** that determine whether a website is accessible to AI agents and automated HTTP clients
- Defines a **scoring methodology** that aggregates check results into a single 0–100 score and a three-level readiness ladder (integer levels 1, 2, 3)
- Defines a **conformance contract** — what an implementing scanner MUST and SHOULD do to be considered conformant with the Agent-Adoption Specification
- Bundles a **B2B SaaS profile** as the first implementation profile (other profiles for e-commerce, media, government, etc., may ship in future versions)

## What this specification does NOT do

- Does NOT specify how individual LLMs rank brands (those algorithms are not publicly documented and not in scope)
- Does NOT make causal claims linking agent-readiness to LLM citation outcomes — the [companion correlation research](https://github.com/respectarium/agent-adoption-research) tests those claims empirically and finds modest, not dramatic, effects
- Does NOT mandate any specific implementation technology (you may use any HTTP client, programming language, or runtime to build a conforming scanner)
- Does NOT define UI/UX conventions for presenting scanner results to humans
- Does NOT impose pricing or commercial terms on any implementer

---

## Status

This specification is **open**. Anyone may implement a scanner against it. To date, the following implementations exist:

- **Respectarium agent-adoption Check tool** (proprietary commercial implementation; the version used to produce the [2026-04 correlation study](https://github.com/respectarium/agent-adoption-research/releases/tag/study-2026-04) is documented in the study release notes)

We invite additional independent implementations. Cross-implementation divergence on shared check definitions is itself a research-relevant phenomenon — see the [correlation study's findings on cross-scanner agreement](https://respectarium.com/research/correlation-2026-04) for empirical context.

---

## How to use this repository

### Read the specification

The canonical specification text lives in [`SPEC.md`](SPEC.md). The same text is rendered as a human-readable web page at [`respectarium.com/spec/agent-adoption/v1`](https://respectarium.com/spec/agent-adoption/v1).

The two views are kept in sync — edits land in this repository first via Pull Request, then the live page is updated. When in doubt, the SPEC.md in the latest tagged release is authoritative.

### Implement a scanner against it

The specification is implementation-agnostic. To build a conforming scanner:

1. Read [`SPEC.md`](SPEC.md), particularly the per-check definitions and conformance section
2. Implement an HTTP client that exercises the checks against target domains
3. Apply the scoring formula and level-gate logic as specified
4. Produce output conforming to the canonical JSON schema at [`schemas/output-v1.schema.json`](schemas/output-v1.schema.json) (described in [`SPEC.md`](SPEC.md) §10)
5. Optionally, run your scanner against the 908 brand domains in the [published study dataset](https://github.com/respectarium/agent-adoption-research/tree/study-2026-04/data) and compare your per-check results to the published reference outputs (cross-scanner divergence is itself a documented finding — see the correlation study)

### Contribute to the specification

We welcome contributions — methodology refinements, new check proposals, conformance clarifications, and corrections.

- For **non-developers:** open an [Issue](../../issues/new/choose) using one of the templates
- For **developers:** open a Pull Request against `main` with your proposed changes; see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines

The decision process for accepting changes is in [`governance/maintainers.md`](governance/maintainers.md).

---

## Versioning

- **v1.0.x** — current line; editorial fixes, clarifications without changing behavior
- **v1.1.0** — additive changes (new optional checks, new profiles)
- **v2.0.0** — breaking changes (renamed checks, score-formula changes, removed checks, new gate criteria)

Each tagged release is **immutable** once published. Errata for past tags are documented in [`CHANGELOG.md`](CHANGELOG.md) with pointers to the corrected analysis in subsequent tags.

The **next major version (v2.0)** is planned for ~Q3 2026, calibrated against findings from the quarterly [Agent-Adoption Correlation Study](https://github.com/respectarium/agent-adoption-research). v2 will recalibrate check weights based on empirical signal-to-outcome relationships established in those studies.

---

## License

This specification is licensed under the **Creative Commons Attribution 4.0 International License (CC-BY 4.0)**.

You may copy, distribute, modify, and build upon this work for any purpose, including commercial, **provided you give appropriate credit to Respectarium**.

Suggested attribution:

> Source: Agent-Adoption Specification by Respectarium — https://respectarium.com/spec/agent-adoption/v1

Full license text: [`LICENSE`](LICENSE)

---

## Citation

When citing this specification in research, implementations, or derived work:

> Agent-Adoption Specification, Version 1.0. Respectarium, 2026-04-26.
> Available at: https://respectarium.com/spec/agent-adoption/v1
> Source: https://github.com/respectarium/agent-adoption-spec/releases/tag/v1.0.0

---

## Companion artifacts

This specification is part of a broader research and methodology program at Respectarium:

- **[Agent-Adoption Correlation Study](https://github.com/respectarium/agent-adoption-research)** — quarterly empirical research testing which signals defined in this specification correlate with LLM visibility outcomes. The Q1 study (`study-2026-04`) is the foundation for evidence-based v2 design.
- **[`respectarium.com/research/correlation-2026-04`](https://respectarium.com/research/correlation-2026-04)** — the formal preprint of the Q1 correlation study
- **Respectarium's hosted implementation** — a hosted scanner conforming to this specification; available at [respectarium.com](https://respectarium.com)

---

## Maintainers

Maintained by Respectarium. See [`governance/maintainers.md`](governance/maintainers.md) for the decision process and contact information.

For private inquiries: open a [Discussion](../../discussions) on this repository.
