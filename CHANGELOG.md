# Changelog

All notable changes to the Agent-Adoption Specification per release.

## [v1.0.0] — 2026-04-26

Initial public release of the Agent-Adoption Specification.

### Specification scope

- 25 measurable agent-readiness checks across 4 categories: `discoverability`, `accessControl`, `contentReadability`, `agentEndpoints`
- 12 scored checks (weight > 0) and 13 informational checks (weight = 0); sum of scored weights = 80
- Three-value status enum: `pass`, `fail`, `neutral` (no `error`, no `skipped`)
- Three readiness levels (integer values 1, 2, 3) with single-check gates: `content-signals` (L1→L2) and `markdown-negotiation` (L2→L3)
- Three named cluster verdicts (`htmlPath`, `spaRenderingCap`, `noViablePathCap`) emitted as a top-level `clusters` array, with normative cluster math (composition order: coefficients → base score → caps)
- v1.0 wire surface is closed at the root (no top-level vendor namespace); vendor extensions live under per-check `details` and `meta.counters`, both open via `additionalProperties: true`
- One bundled profile: `b2b-saas` ([`profiles/b2b-saas.json`](profiles/b2b-saas.json))
- Normative output schema at [`schemas/output-v1.schema.json`](schemas/output-v1.schema.json) (JSON Schema draft 2020-12)
- Score formula: `round(100 * passWeight / (passWeight + failWeight))`; `neutral` checks and `weight: 0` checks excluded; divide-by-zero short-circuits to `score: 0`
- Conformance contract for v1.0 implementations (see [`SPEC.md`](SPEC.md) §11)

### Companion artifacts

- Live specification page: https://respectarium.com/spec/agent-adoption/v1
- Companion empirical research (Q1 baseline): https://respectarium.com/research/correlation-2026-04
- Reference data and analysis: https://github.com/respectarium/agent-adoption-research/releases/tag/study-2026-04

### License

Licensed under [CC-BY 4.0](LICENSE) (free use, modification, redistribution with attribution to Respectarium).

[v1.0.0]: https://github.com/respectarium/agent-adoption-spec/releases/tag/v1.0.0
