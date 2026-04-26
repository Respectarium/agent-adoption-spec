# Agent-Adoption Specification, Version 1.0

**Status:** Approved for v1.0.0 release.
**Date:** 2026-04-26
**Authors:** Respectarium
**License:** [Creative Commons Attribution 4.0 International (CC-BY 4.0)](LICENSE)
**Canonical URL:** https://respectarium.com/spec/agent-adoption/v1
**Source repository:** https://github.com/respectarium/agent-adoption-spec

---

## Abstract

The Agent-Adoption Specification defines an open methodology for measuring how observable a website's agent-readiness signals are to AI agents and other automated HTTP clients. It specifies a fixed catalog of **25 checks** organized into **4 categories**, a **three-status verdict enum** (`pass` / `fail` / `neutral`), a deterministic **integer scoring model** producing a 0–100 aggregate score, a gate-driven **three-level readiness ladder** (1, 2, 3), and a normative **JSON output schema** governing the wire format. The specification is implementation-agnostic: any scanner that emits a JSON document validating against `schemas/output-v1.schema.json`, applies the scoring formula and cluster math defined herein, and adheres to the conformance rules in [§11](#11-conformance) is conformant.

---

## Notational conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals, as shown here.

Conformance requirements are stated inline within individual check definitions ([§5](#5-checks-catalog)) and consolidated in the Conformance section ([§11](#11-conformance)).

Code spans (`like-this`) denote literal field names, enum values, JSON property names, or check identifiers as they appear on the wire. Capitalized terms (e.g., **Specification**, **Profile**, **Cluster**) are defined in the Terminology section ([§2](#2-terminology)).

---

## Table of contents

1. [Overview & purpose](#1-overview--purpose)
2. [Terminology](#2-terminology)
3. [Scope & non-goals](#3-scope--non-goals)
4. [Categories](#4-categories)
5. [Checks (catalog)](#5-checks-catalog)
6. [Status](#6-status)
7. [Scoring](#7-scoring)
8. [Levels](#8-levels)
9. [Profiles](#9-profiles)
10. [Output format](#10-output-format)
11. [Conformance](#11-conformance)
12. [Versioning](#12-versioning)
13. [Annex A — Defensive framing](#annex-a--defensive-framing)
14. [Annex B — Worked example](#annex-b--worked-example)
15. [Annex C — Implementation notes](#annex-c--implementation-notes)

---

## 1. Overview & purpose

The Agent-Adoption Specification (the "Specification" or "spec") defines how a website signals readiness to be consumed by autonomous agents — search bots, AI assistants, retrieval systems, and protocol-aware clients. It defines a fixed catalog of 25 observable HTTP-level checks across four categories, a deterministic scoring model, a three-level readiness ladder, and a JSON output schema. Conforming implementations ("scanners") accept a domain as input and emit a single self-contained JSON document conforming to the wire schema in [`schemas/output-v1.schema.json`](schemas/output-v1.schema.json).

The Specification is written for three audiences:

- **Consumers** of scan output — analytics tools, search-readiness dashboards, agentic platforms reading scan JSON to make routing decisions.
- **Implementers** of scanners — organizations building tools that emit conforming output.
- **Brand owners** — operators of websites who want a stable definition of what "agent-ready" means.

The 25 checks each correspond to a specific external standard, vendor convention, or measurable HTTP behavior. The Specification does not invent semantics; it codifies measurement.

### 1.1 Defensive framing

The Specification measures **agent-readiness signals**, not predicted AI traffic, conversion, brand mention frequency, or search-result rank. A high score does not guarantee that any particular agent will visit, cite, or correctly interpret the site; it indicates that the standardized signals an agent would expect to find are present and well-formed. Conversely, a low score does not predict invisibility — many agents tolerate missing signals via heuristics. The Specification is a measurement contract, not a marketing promise.

This framing carries through every part of the Specification: status verdicts, scores, levels, and cluster verdicts describe **observable site state**, not predicted behavior of any agent. Annex A elaborates.

### 1.2 Relationship to companion research

This Specification is descriptive (defining what a conforming scanner measures and how) rather than predictive. Empirical findings about which signals correlate with downstream LLM visibility outcomes are reported in the companion [Agent-Adoption Correlation Study](https://respectarium.com/research/correlation-2026-04), which uses this Specification as its measurement protocol.

A v2.0 update is planned for approximately Q3 2026, calibrated against findings from quarterly correlation re-runs. Tagged releases are immutable; v1.0 and any future major versions remain accessible at their respective version-suffixed URLs. See [§12](#12-versioning) for the full versioning policy.

---

## 2. Terminology

This section defines the terms used throughout the Specification. Implementations and downstream content **SHOULD** use these terms with the meanings given here.

- **Specification** (or "spec") — This document plus the JSON schemas and profile JSON it references. Normative authority for what a conforming implementation MUST emit.
- **Scanner** (or "Conforming scanner") — Any software implementation that accepts a domain as input and emits JSON output validating against `schemas/output-v1.schema.json`, applies the scoring formula and cluster math defined in this document, and adheres to the conformance rules in [§11](#11-conformance).
- **Implementer** — A party building a scanner that conforms to this Specification.
- **Brand** — The website being measured. The Specification operates on a single root domain (eTLD+1) per scan.
- **Profile** — A bundled JSON document declaring which checks are active for a given vertical or use case, plus per-check weight overrides if any. v1.0 ships exactly one profile (`b2b-saas`). v1.0 does NOT define a custom-profile mechanism; future versions may. See [§9](#9-profiles).
- **Check** — One of 25 named, atomic measurements. Each check has a stable kebab-case identifier, belongs to exactly one category, carries an integer weight, and emits exactly one status per scan plus a structured `details` payload. The full list is in [§5](#5-checks-catalog).
- **Score** — An integer 0–100 derived deterministically from the per-check statuses and weights via the formula in [§7](#7-scoring). Score is computed both at the top level (across all scored checks) and per category (scoped to that category's scored checks).
- **Level** — An integer (1, 2, or 3) representing site readiness tier. v1.0 defines exactly three levels; future versions may add more. See [§8](#8-levels).
- **Level Gate** — A check whose `pass` status is required to advance from one level to the next. v1.0 defines two gates: `content-signals` (L1→L2) and `markdown-negotiation` (L2→L3). Level is gate-driven, NOT score-driven; a site MAY have a low score but reach L2 if the gate passes, or vice versa.
- **Status** — The per-check verdict, exactly one of `pass`, `fail`, `neutral`. Defined in [§6](#6-status).
- **Weight** — An integer drawn from the closed set `{0, 4, 5, 6, 7, 8, 10}` declaring how much a check contributes to scoring. Weight `0` means informational (not scored); weights `4`–`10` are scored. Sum of all scored weights in v1.0 = 80.
- **Scored vs Informational** — A check is "scored" when its weight > 0 (and `scored: true` on the wire); "informational" when weight = 0 (and `scored: false`). v1.0 has 12 scored checks and 13 informational checks. Informational checks still emit a status and `details` payload but contribute nothing to the score; some informational checks are level gates.
- **Cluster** — A named cross-check rule that adjusts the final score based on a combination of check results. v1.0 defines exactly three clusters (`htmlPath`, `spaRenderingCap`, `noViablePathCap`) emitted in fixed order as the top-level `clusters[]` array. Clusters are part of the score-computation discipline; the wire shape lives at the top level. See [§7.3](#73-cluster-math).
- **Cluster verdict** — A single entry in the top-level `clusters` array describing whether one named cluster's effect was applied during scoring, plus its parameters.
- **Coefficient cluster** (`kind: "coefficient"`) — A cluster that multiplies one or more per-check weights before scoring. v1.0 defines one (`htmlPath`).
- **Cap cluster** (`kind: "cap"`) — A cluster that clamps the final top-level score to a maximum value. v1.0 defines two (`spaRenderingCap`, `noViablePathCap`). When multiple cap clusters trigger, the lowest cap wins (caps do not compound).
- **Cluster math** — The composition rule by which clusters affect the final score: coefficient-kind clusters multiply per-check weights first, base score is computed, then cap-kind clusters clamp the final top-level score. Per-category scores are NOT subject to cluster math. See [§7.3](#73-cluster-math).
- **Wire Format** — The on-the-wire JSON encoding rules. v1.0 uses **camelCase** for all object keys and category enum values, **kebab-case** for check IDs, ISO 8601 with `Z` suffix for timestamps, and integer types for `level`, `score`, `weight`, all duration fields, and all category counts.
- **Vendor extension surfaces** — v1.0 does NOT define a top-level vendor namespace. Implementations that need to emit vendor-specific data have two open surfaces: the per-check `details` object (`additionalProperties: true`, except `details.body` is forbidden — see [§10.2](#102-per-check-object-shape-11-fields) and [§11.3](#113-must-not-requirements)), and the `meta.counters` object (open at both levels — see [§10.5](#105-meta-object-shape)). A future minor version MAY introduce a top-level vendor namespace if implementer demand emerges; v1.0 deliberately omits it to keep the wire surface minimal.
- **`nextLevel`** — An optional top-level object describing the next level the site can attempt and the check IDs that must flip to `pass` to advance. **Omitted (not nullified)** when the site is at the highest defined level (v1.0: level 3). See [§8](#8-levels).
- **`details.kind`** — A string field that, when present inside a check's `details` payload, MUST equal that check's `id`. Allows consumers to discriminate per-check structured payloads via duck-typing without re-deriving the field from context. The schema does not require it; implementations that emit it MUST set it equal to the check ID.
- **Categories** — The four top-level groupings of checks. Defined in [§4](#4-categories).
  - `discoverability` — Can agents find the navigation signals they need?
  - `accessControl` — Has the site declared its position on AI use of its content?
  - `contentReadability` — Can fetched content actually be consumed?
  - `agentEndpoints` — Does the site expose programmatic surfaces for agents?

---

## 3. Scope & non-goals

### 3.1 v1.0 covers

- A fixed catalog of **25 checks** organized into **4 categories**, with stable kebab-case IDs and stable integer weights ([§4](#4-categories), [§5](#5-checks-catalog)).
- A single bundled **profile** (`b2b-saas`) declaring the active checks and weights ([§9](#9-profiles)).
- A normative **JSON output schema** ([`schemas/output-v1.schema.json`](schemas/output-v1.schema.json)) validating a single self-contained scan response (13 top-level fields).
- A deterministic **scoring formula** with explicit edge-case behavior ([§7](#7-scoring)).
- A **three-level readiness ladder** with two named gates ([§8](#8-levels)).
- Three named **cluster verdicts** with their trigger conditions and effects on score ([§7.3](#73-cluster-math)).
- Defensive framing language clarifying that the spec measures readiness signals, not predicted AI traffic ([Annex A](#annex-a--defensive-framing)).
- Conformance rules (MUST / MAY / MUST NOT) ([§11](#11-conformance)).

### 3.2 v1.0 does NOT cover

- **No `error` status.** The status enum is exactly three values. Implementations encountering a runtime failure within a single check MUST map it to one of the three defined statuses (see [§6](#6-status)). v2.0+ MAY introduce a fourth status.
- **No `tier` abstraction.** Weights are integers in a closed set; the spec does not define named tiers (Critical / High / Medium / Low) on the wire. Vendor descriptions MAY group weights into tiers for human consumption, but the wire MUST emit raw integers.
- **No formal profile schema.** v1.0 ships one profile JSON; the structure of profile JSON is illustrative, not normatively schema-validated. v2.0+ MAY introduce `profile-v1.schema.json`.
- **No custom-profile mechanism.** v1.0 implementations MUST emit `profile: "b2b-saas"`. Custom profiles are deferred.
- **No localization.** All check `description` and `message` strings are English. Localization layers are out of scope for v1.0.
- **No signed responses.** Scan output is not signed; consumers wanting attestation must implement their own signing layer.
- **No browser fingerprint or render-time JS analysis** beyond what `rendering-strategy` performs (a single before/after-render content-length comparison).
- **No active probes that mutate site state.** All checks are read-only.
- **No throttling or rate-limit specification.** Scanners decide their own polling cadence subject to robots-respecting behavior.
- **No conformance test suite.** A formal test suite is deferred. Implementations MAY use any conforming output as a validation target for their own schema-compliance tests.

### 3.3 What v2.0+ may add

(Specifics intentionally deferred to avoid pre-committing v2.0 design.)

- An `error` status value distinct from `fail` and `neutral`.
- A formal profile schema and a custom-profile mechanism.
- Additional checks, additional categories, or additional level tiers.
- A normative localization layer.
- Signed/attested scan output.

---

## 4. Categories

v1.0 defines exactly four categories, in this canonical order. Conforming scanners MUST emit `categoryReports` as an array of length 4 in this order; see [§10](#10-output-format) for the full output shape.

### 4.1 `discoverability`

**Description:** Can agents find the signals they need to navigate the site? Covers `robots.txt`, XML sitemap, and `Link:` response headers — the first three signals any crawler reads before parsing HTML.

**Weight subtotal:** 11

| id                  | weight | scored |
| ------------------- | ------ | ------ |
| `robots-txt-exists` | 7      | true   |
| `sitemap-exists`    | 4      | true   |
| `link-headers`      | 0      | false  |

### 4.2 `accessControl`

**Description:** Is the site telling AI systems what they may and may not do with its content? Covers per-bot `robots.txt` rules, content-usage signals (Content-Signal / AIPREF), cryptographic bot authentication, and blanket-allow posture detection.

**Weight subtotal:** 7

| id                 | weight | scored | gate      |
| ------------------ | ------ | ------ | --------- |
| `ai-bot-rules`     | 7      | true   | —         |
| `content-signals`  | 0      | false  | **L1→L2** |
| `web-bot-auth`     | 0      | false  | —         |
| `robots-allow-all` | 0      | false  | —         |

### 4.3 `contentReadability`

**Description:** Once an agent is inside, can it actually read what it fetches? Covers markdown availability (per-URL twins and Accept-header negotiation), page size relative to context windows, rendering strategy, HTTP status-code honesty, redirect behavior, llms.txt shape, AGENTS.md presence, and cache header hygiene.

**Weight subtotal:** 39

| id                              | weight | scored | gate      |
| ------------------------------- | ------ | ------ | --------- |
| `rendering-strategy`            | 10     | true   | —         |
| `markdown-url-support`          | 8      | true   | —         |
| `http-status-codes`             | 6      | true   | —         |
| `page-size-html`                | 6      | true   | —         |
| `markdown-negotiation`          | 5      | true   | **L2→L3** |
| `redirect-behavior`             | 4      | true   | —         |
| `agents-md-detection`           | 0      | false  | —         |
| `cache-header-hygiene`          | 0      | false  | —         |
| `llms-txt-exists`               | 0      | false  | —         |
| `llms-txt-has-optional-section` | 0      | false  | —         |
| `llms-txt-size`                 | 0      | false  | —         |
| `llms-txt-valid`                | 0      | false  | —         |

### 4.4 `agentEndpoints`

**Description:** Does the site expose the surfaces agents need to call it programmatically? Covers MCP Server Cards, A2A Agent Cards, Agent Skills indexes, OAuth Authorization Server / OpenID Connect discovery, OAuth Protected Resource metadata, and `/.well-known/api-catalog`.

**Weight subtotal:** 23

| id                         | weight | scored |
| -------------------------- | ------ | ------ |
| `mcp-server-card`          | 10     | true   |
| `oauth-protected-resource` | 7      | true   |
| `oauth-discovery`          | 6      | true   |
| `a2a-agent-card`           | 0      | false  |
| `agent-skills`             | 0      | false  |
| `api-catalog`              | 0      | false  |

### 4.5 Totals

**25 checks across 4 categories. Sum of all weights = 11 + 7 + 39 + 23 = 80.** 12 scored checks (weight > 0), 13 informational checks (weight = 0).

---

## 5. Checks (catalog)

The 25 checks in canonical category order, with checks within each category sorted by weight (descending) then ID (alphabetic). Trigger conditions are normative: conforming scanners MUST classify status as defined here. External standards URLs are non-normative pointers — implementations MAY include any subset in the per-check `specUrls` array on the wire.

The per-check trigger language uses the active voice ("Returns `pass` when ...") to denote what a conforming implementation MUST do. Where a trigger references probing a URL, the implementation MAY choose its own request headers, retry policy, and timeout values unless explicitly constrained.

---

### 5.1 `discoverability`

#### `robots-txt-exists`

- **Category:** `discoverability`
- **Weight:** 7 (scored)
- **What it tests:** Whether `/robots.txt` is reachable and parseable as a robots-exclusion file per [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309).
- **Returns `pass`** when `GET /robots.txt` returns a 2xx status, the response `Content-Type` begins with `text/`, and the body contains at least one `User-agent:` directive line.
- **Returns `fail`** when the response is non-2xx, the content-type is not `text/*`, the body is empty, the body lacks any `User-agent:` directive, or the fetch fails at the network level.
- **Returns `neutral`:** This check has no neutral case; fetch failures map to `fail`.
- **Notes:** Several access-control checks (`ai-bot-rules`, `content-signals`, `robots-allow-all`) declare a hard dependency on this check. When this check does not pass, those dependents emit `neutral` with a dependency-skip message.

#### `sitemap-exists`

- **Category:** `discoverability`
- **Weight:** 4 (scored)
- **What it tests:** Whether at least one valid XML sitemap is reachable, discovered first via `Sitemap:` directives in `robots.txt`, then falling back to `/sitemap.xml`. See [sitemaps.org protocol](https://www.sitemaps.org/protocol.html).
- **Returns `pass`** when at least one candidate URL returns a 2xx response that parses as XML with a root element of `<urlset>` or `<sitemapindex>`.
- **Returns `fail`** when all candidates were probed, at least one was reachable, and none parsed to a valid sitemap structure.
- **Returns `neutral`** when all candidates failed at the network level, or every reachable candidate's body exceeded an implementation-defined pre-parse size cap (a conservative default of 2 MB is RECOMMENDED).

#### `link-headers`

- **Category:** `discoverability`
- **Weight:** 0 (informational)
- **What it tests:** Whether the homepage emits `Link:` response headers exposing agent-relevant resources (API catalogs, alternates, service docs) per [RFC 8288](https://datatracker.ietf.org/doc/html/rfc8288).
- **Returns `pass`** when a `Link:` header is present on the homepage AND at least one rel value is something other than browser-presentation rels (`preload`, `prefetch`, `stylesheet`, `icon`, `dns-prefetch`, `preconnect`, `modulepreload`, `apple-touch-icon`, or `unknown`).
- **Returns `fail`** when a `Link:` header is present but every rel value is browser-presentation only (or rel-less).
- **Returns `neutral`** when no `Link:` header is present on the homepage.

---

### 5.2 `accessControl`

#### `ai-bot-rules`

- **Category:** `accessControl`
- **Weight:** 7 (scored)
- **What it tests:** Whether `robots.txt` declares per-AI-bot rules or content-usage directives.
- **Returns `pass`** when the robots body contains at least one per-`User-agent` block targeting a known AI crawler with at least one `Allow:` or `Disallow:` rule, OR the body contains at least one `Content-Signal:` (legacy) or `Content-Usage:` (AIPREF) directive with a non-empty value.
- **Returns `fail`** when the robots body is present and parseable, but neither per-AI-bot rules nor a Content-Signal / Content-Usage directive is found.
- **Returns `neutral`** when the dependency `robots-txt-exists` did not pass (dependency-skip), or the robots body is empty.
- **Relevant standards:** [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309), [Cloudflare Content-Signal](https://developers.cloudflare.com/bots/additional-configurations/content-signals/), [IETF AIPREF Vocabulary](https://datatracker.ietf.org/doc/draft-ietf-aipref-vocab/).
- **Notes:** Hard `dependsOn: ["robots-txt-exists"]`.

#### `content-signals`

- **Category:** `accessControl`
- **Weight:** 0 (informational)
- **Level gate:** **L1 → L2.** The site advances from Level 1 to Level 2 when this check returns `pass`.
- **What it tests:** Whether `robots.txt` carries at least one Content-Signal (Cloudflare) or AIPREF Content-Usage directive declaring permitted post-fetch use of the content.
- **Returns `pass`** when the robots body contains at least one `Content-Signal:` or `Content-Usage:` directive line. Token-level recognition is forward-tolerant.
- **Returns `fail`:** This check has no fail case. Verdicts other than pass map to `neutral`.
- **Returns `neutral`** when the robots body is empty, or no Content-Signal / Content-Usage directives are present, or the dependency `robots-txt-exists` did not pass.
- **Relevant standards:** [Cloudflare Content-Signal](https://developers.cloudflare.com/bots/additional-configurations/content-signals/), [IETF AIPREF Vocabulary](https://datatracker.ietf.org/doc/draft-ietf-aipref-vocab/), [IETF AIPREF Attach](https://datatracker.ietf.org/doc/draft-ietf-aipref-attach/).
- **Notes:** Hard `dependsOn: ["robots-txt-exists"]`. Although informational (weight 0), this check is the **L1→L2 level gate**.

#### `web-bot-auth`

- **Category:** `accessControl`
- **Weight:** 0 (informational)
- **What it tests:** Whether the site publishes a signing-key directory at `/.well-known/http-message-signatures-directory` for HTTP Message Signatures-based bot authentication. Presence signals that the site operates an AI crawler whose identity targets can verify.
- **Returns `pass`** when `GET /.well-known/http-message-signatures-directory` returns a 2xx response with parseable JSON containing a `keys` array of objects, each with a non-empty `kty` string.
- **Returns `fail`** when the response is non-2xx, the body is unparseable JSON, the `keys` array is missing or empty, or any key lacks a `kty` field.
- **Returns `neutral`** when the endpoint is unreachable at the network level.
- **Relevant standards:** [HTTP Message Signatures Directory draft](https://datatracker.ietf.org/doc/html/draft-meunier-http-message-signatures-directory-05), [Web Bot Auth Architecture draft](https://datatracker.ietf.org/doc/draft-meunier-web-bot-auth-architecture/).

#### `robots-allow-all`

- **Category:** `accessControl`
- **Weight:** 0 (informational)
- **What it tests:** Whether `robots.txt` declares a blanket-allow posture: a wildcard `User-agent: *` group that does not block, AND no other UA group containing a blanket `Disallow: /`.
- **Returns `pass`** when a `User-agent: *` group is present AND that group either contains an explicit `Allow: /` or has no `Disallow:` rules, AND no other UA group contains `Disallow: /`.
- **Returns `fail`** when the robots body is present but does not satisfy the wildcard-permissive + no-cross-bot-blanket-block conditions.
- **Returns `neutral`** when the robots body is empty (dependency-skip path), or the dependency `robots-txt-exists` did not pass.
- **Relevant standards:** [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309).
- **Notes:** Hard `dependsOn: ["robots-txt-exists"]`.

---

### 5.3 `contentReadability`

#### `rendering-strategy`

- **Category:** `contentReadability`
- **Weight:** 10 (scored)
- **What it tests:** Classifies the homepage's rendering strategy by comparing the plain HTML body to a headlessly-rendered version. Determines whether agents can read meaningful content without executing JavaScript.
- **Returns `pass`** when the plain (non-rendered) body is NOT detected as an SPA shell, the headless render succeeds, AND the rendered/plain content-length ratio is < 1.05 (full server-side rendering) or between 1.05 and 1.20 inclusive (server-side rendering with partial hydration).
- **Returns `fail`** when the plain body is detected as an SPA shell (immediate short-circuit); OR when plain is empty and rendered is empty; OR when plain is empty and rendered has content; OR when the rendered/plain ratio exceeds 1.20.
- **Returns `neutral`** when the headless rendering capability is unavailable AND the plain body is not an SPA shell.
- **Notes:** **Scoring interaction:** when this check fails, the `htmlPath` cluster triggers (zeroing out `page-size-html`'s weight) AND the `spaRenderingCap` cluster triggers (capping the final top-level score at 39 or 59 depending on severity). See [§7.3](#73-cluster-math).

#### `markdown-url-support`

- **Category:** `contentReadability`
- **Weight:** 8 (scored)
- **What it tests:** Whether the site provides `.md` twins for HTML pages (e.g., `/page.md` alongside `/page.html`), giving agents a markdown fetch path with no HTML wrappers, ads, or modals. Implementations sample up to 5 URLs (sitemap pool first; homepage fallback).
- **Returns `pass`** when the number of measurable probes (served + missing) is at least 3, AND the ratio of served (2xx + markdown-shaped body) to measurable is at least 0.5.
- **Returns `fail`** when measurable probes ≥ 3 AND served/measurable < 0.5.
- **Returns `neutral`** when no samples could be collected at all; OR at least half of probe attempts were unmeasurable (blocked, errored); OR fewer than 3 measurable probes were obtained (insufficient sample).
- **Relevant standards:** [llms.txt convention](https://llmstxt.org/), [Cloudflare markdown-for-agents](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/).

#### `http-status-codes`

- **Category:** `contentReadability`
- **Weight:** 6 (scored)
- **What it tests:** Whether the site honestly returns HTTP 4xx for missing pages instead of returning 200 with a "page not found" body (a "soft 404" that pollutes agent caches with garbage canonical content).
- **Returns `pass`** when a synthetic probe to a URL designed to not exist on the target site (e.g., a randomized path under a sentinel prefix) returns an HTTP 4xx status.
- **Returns `fail`** when the probe returns a 2xx status AND the body is not detectably an SPA shell (i.e., a soft-404 has been confirmed).
- **Returns `neutral`** when the probe fails at the network level; OR returns 3xx (indeterminate); OR returns 5xx; OR returns 1xx; OR returns 2xx with an SPA-shell body (the SPA may legitimately handle 404 client-side).

#### `page-size-html`

- **Category:** `contentReadability`
- **Weight:** 6 (scored)
- **What it tests:** Measures how much markdown content each page would deliver to an agent's context window. Implementations sample up to 10 URLs and convert each page's HTML to markdown; per-page pass = converted output ≤ 50,000 characters.
- **Returns `pass`** when successful samples ≥ 3, AND `pass count * 2 > successful count` (clear majority pass), AND `fail count == 0` (no oversize pages).
- **Returns `fail`** when there is a failure majority (`fail count * 2 > successful count`); OR a mixed warn+fail outcome (conservative fail).
- **Returns `neutral`** when no samples could be collected; OR fewer than 3 successful samples were obtained.
- **Notes:** **Scoring interaction:** when `rendering-strategy` fails, the `htmlPath` coefficient cluster zeros out this check's effective weight (excluding it from score). The check still emits its normal verdict; the cluster effect is applied during scoring, not by suppressing the check itself. See [§7.3](#73-cluster-math).

#### `markdown-negotiation`

- **Category:** `contentReadability`
- **Weight:** 5 (scored)
- **Level gate:** **L2 → L3.** The site advances from Level 2 to Level 3 when this check returns `pass`.
- **What it tests:** Whether the site honors `Accept: text/markdown` content negotiation on its homepage URL — serving HTML to humans and markdown to agents from a single URL (no duplicate-URL strategy).
- **Returns `pass`** when a probe of the origin URL with `Accept: text/markdown` returns a 2xx status, AND either the response `Content-Type` includes `text/markdown` and the body is not a full HTML document, OR the body is markdown-shaped and not HTML-shaped.
- **Returns `fail`** when the probe returns 2xx with an HTML response; OR returns a non-2xx status in the ranges 100–199, 300–399 (excluding 5xx — see neutral), or 400–499 (excluding 403 and 429 — see neutral).
- **Returns `neutral`** when the endpoint is unreachable at the network level; OR the redirect was blocked; OR the response was 403 or 429 (treated as WAF interference); OR the response was ≥ 500.
- **Relevant standards:** [llms.txt convention](https://llmstxt.org/), [Cloudflare markdown-for-agents](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/).

#### `redirect-behavior`

- **Category:** `contentReadability`
- **Weight:** 4 (scored)
- **What it tests:** Whether the site uses agent-friendly redirect mechanisms (same-domain HTTP 3xx) versus agent-hostile ones (cross-domain redirects, JavaScript-only redirects). Implementations sample up to 5 URLs.
- **Returns `pass`** when successful samples ≥ 3 AND no failed samples (where each per-URL pass means the URL did not redirect, or redirected within the same eTLD+1).
- **Returns `fail`** when successful samples ≥ 3 AND at least one failed sample (cross-eTLD redirect or JavaScript redirect detected).
- **Returns `neutral`** when no samples could be collected; OR fewer than 3 successful samples were obtained.

#### `agents-md-detection`

- **Category:** `contentReadability`
- **Weight:** 0 (informational)
- **What it tests:** Whether the site publishes an `/AGENTS.md` file (a coding-agent convention; presence alone is the signal).
- **Returns `pass`** when `GET /AGENTS.md` returns a 2xx status with a `Content-Type` of `text/markdown`, `text/plain`, or `text/x-markdown` (charset suffix stripped), AND the body is non-empty, AND the body does not begin with HTML markers, AND the first line does not contain "page not found", "404", or "not found", AND the body is at least 50 bytes.
- **Returns `fail`** when the endpoint is reachable but any of the above predicates fails.
- **Returns `neutral`** when the endpoint is unreachable at the network level.
- **Notes:** This check follows a community convention without a published RFC; `specUrls` is typically empty.

#### `cache-header-hygiene`

- **Category:** `contentReadability`
- **Weight:** 0 (informational)
- **What it tests:** Whether the homepage emits at least one of the standard cache-validation response headers (`Cache-Control`, `ETag`, `Last-Modified`), enabling efficient agent re-fetching per [RFC 7234](https://www.rfc-editor.org/rfc/rfc7234).
- **Returns `pass`** when at least one of `Cache-Control`, `ETag`, or `Last-Modified` is present in the homepage response headers (case-insensitive lookup).
- **Returns `fail`** when the homepage headers are non-empty but none of the three primary headers are present.
- **Returns `neutral`** when the homepage header map is empty (defensively handled; in practice unreachable).

#### `llms-txt-exists`

- **Category:** `contentReadability`
- **Weight:** 0 (informational)
- **What it tests:** Whether an `llms.txt` file is reachable, probing `/llms.txt` first then `/docs/llms.txt`. Verdict is informational and structural details are carried in the per-check `details` payload (specifically a `discoveredFiles` listing).
- **Returns `pass`:** This check has no pass case. Status is always `neutral`; verdict shape is carried in `details`.
- **Returns `fail`:** This check has no fail case. Status is always `neutral`; verdict shape is carried in `details`.
- **Returns `neutral`** always. The `details` payload distinguishes the discovered-vs-not-discovered cases. Downstream `llms-txt-*` checks read this check's cached body via dependency.
- **Relevant standards:** [llms.txt convention](https://llmstxt.org/).

#### `llms-txt-has-optional-section`

- **Category:** `contentReadability`
- **Weight:** 0 (informational)
- **What it tests:** Whether the discovered `llms.txt` body contains an `## Optional` H2 section.
- **Returns `pass`** when a cached `llms.txt` body is present AND the body matches the regex `/^##\s+Optional\b/im` (an H2 line whose first word starts with "Optional").
- **Returns `fail`** when a body is present but no `## Optional` H2 section is found.
- **Returns `neutral`** when no `llms.txt` body was cached upstream (dependency-skip from `llms-txt-exists`).
- **Relevant standards:** [llms.txt convention](https://llmstxt.org/).

#### `llms-txt-size`

- **Category:** `contentReadability`
- **Weight:** 0 (informational)
- **What it tests:** Reports the byte size of the discovered `llms.txt` against context-window-fit tiers. Verdict is informational and tier classification is carried in `details.tier`.
- **Returns `pass`:** This check has no pass case. Status is always `neutral`.
- **Returns `fail`:** This check has no fail case. Status is always `neutral`.
- **Returns `neutral`** always. The `details.tier` field carries the verdict: `"pass"` when ≤ 50,000 bytes, `"warn"` when 50,001–100,000 bytes, `"fail"` when > 100,000 bytes. When no file was discovered upstream, returns `neutral` with no `tier`.
- **Relevant standards:** [llms.txt convention](https://llmstxt.org/).

#### `llms-txt-valid`

- **Category:** `contentReadability`
- **Weight:** 0 (informational)
- **What it tests:** Reports the structural shape of the discovered `llms.txt`. Verdict is informational and structural classification is carried in `details.verdict`.
- **Returns `pass`:** This check has no pass case. Status is always `neutral`.
- **Returns `fail`:** This check has no fail case. Status is always `neutral`.
- **Returns `neutral`** always. The `details.verdict` field carries the structural classification: `"structured"` when the first non-empty line is an H1 (`# `) AND the body contains either a blockquote (`> `) line or an `## ` section heading paired with a `[text](https://...)` link; `"minimal"` when only an H1 is present; `"unstructured"` when the first non-empty line is not an H1; `"skipped"` when there is no body. When no body was cached upstream, returns `neutral` with no `verdict`.
- **Relevant standards:** [llms.txt convention](https://llmstxt.org/).

---

### 5.4 `agentEndpoints`

#### `mcp-server-card`

- **Category:** `agentEndpoints`
- **Weight:** 10 (scored)
- **What it tests:** Whether the site advertises a Model Context Protocol (MCP) server card at one of the well-known paths, declaring its MCP endpoint to MCP clients.
- **Returns `pass`** when any of three candidate paths (`/.well-known/mcp/server-card.json`, `/.well-known/mcp/server-cards.json`, `/.well-known/mcp.json`) returns parseable JSON containing a `serverInfo.name` (string) AND `transport.type` (string) AND either `transport.url` or `transport.endpoint` (string).
- **Returns `fail`** when the first parseable JSON found at any candidate path is invalid (does not satisfy the required-field shape); OR when all three paths returned non-network-failure responses but no parseable JSON was found.
- **Returns `neutral`** when all three candidate paths failed at the network level.
- **Relevant standards:** [MCP server-card charter](https://modelcontextprotocol.io/community/server-card/charter); the MCP Server Card discovery proposal is in flight (see [PR #2127](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2127), [PR #2525](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2525)).

#### `oauth-protected-resource`

- **Category:** `agentEndpoints`
- **Weight:** 7 (scored)
- **What it tests:** Whether `/.well-known/oauth-protected-resource` returns valid OAuth 2.0 Protected Resource Metadata per [RFC 9728](https://www.rfc-editor.org/rfc/rfc9728), identifying which authorization server protects the site's API.
- **Returns `pass`** when `GET /.well-known/oauth-protected-resource` returns a 2xx status with parseable JSON containing a well-formed http(s) URL string in the `resource` field.
- **Returns `fail`** when non-2xx status, unparseable JSON, missing `resource` field, or `resource` is not a well-formed URL.
- **Returns `neutral`** when the endpoint is unreachable at the network level.

#### `oauth-discovery`

- **Category:** `agentEndpoints`
- **Weight:** 6 (scored)
- **What it tests:** Whether the site exposes OAuth Authorization Server Metadata ([RFC 8414](https://www.rfc-editor.org/rfc/rfc8414)) at `/.well-known/oauth-authorization-server` or [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) metadata at `/.well-known/openid-configuration` so agents can locate the auth endpoints.
- **Returns `pass`** when either `/.well-known/oauth-authorization-server` OR the fallback `/.well-known/openid-configuration` returns parseable JSON with: an `issuer` (https URL), `response_types_supported` (non-empty array), AND at least one of `authorization_endpoint` or `token_endpoint`.
- **Returns `fail`** when the primary path returns parseable JSON but the contents are invalid (immediate fail); OR the primary path is unparseable AND the fallback returns parseable but invalid JSON; OR both paths returned non-404 responses but neither yielded valid metadata.
- **Returns `neutral`** when both paths cleanly returned 404 (no OAuth surface present at the well-known locations — the check opts out of the score denominator for this case).
- **Notes:** This is the only scored check that can return `neutral` via a deliberate "hedge" path (clean-404-on-both-paths). The check thereby excludes itself from the score denominator when no OAuth surface exists at all, rather than penalizing sites that simply have no auth endpoint to advertise.

#### `a2a-agent-card`

- **Category:** `agentEndpoints`
- **Weight:** 0 (informational)
- **What it tests:** Whether the site publishes an A2A (Agent-to-Agent) Agent Card at `/.well-known/agent-card.json` per the A2A protocol specification.
- **Returns `pass`** when `GET /.well-known/agent-card.json` returns a 2xx response with parseable JSON containing a non-empty `name`, non-empty `description`, non-empty `url`, a `capabilities` object, and a `skills` array.
- **Returns `fail`** when non-2xx status, non-JSON body, root not an object, or any of the required fields missing or invalid.
- **Returns `neutral`** when the endpoint is unreachable at the network level.
- **Relevant standards:** [A2A Protocol Specification](https://a2a-protocol.org/latest/specification/), [A2A Agent Discovery](https://a2a-protocol.org/latest/topics/agent-discovery/).

#### `agent-skills`

- **Category:** `agentEndpoints`
- **Weight:** 0 (informational)
- **What it tests:** Whether the site exposes an Agent Skills Discovery index at `/.well-known/agent-skills/index.json` (or fallback `/.well-known/skills/index.json`).
- **Returns `pass`** when the first parseable JSON of the two candidate paths is an object containing a `$schema` (string) AND a `skills` array, AND the `$schema` matches the canonical 0.2.0 URL or a recognized semver pattern (`^(0\.\d+\.\d+|1\.\d+\.\d+)$`), AND every skill entry has a `digest` matching `/^sha256:[a-f0-9]{64}$/`.
- **Returns `fail`** when the root parses to a non-object; OR the `$schema` or `skills` field is missing; OR a recognized schema is present but digest validation fails on at least one skill; OR no parseable JSON was found across both paths AND not all paths network-failed.
- **Returns `neutral`** when all paths failed at the network level; OR a `$schema` is present but its URL/version is not recognized (forward-compat neutral).
- **Relevant standards:** [Agent Skills Discovery RFC](https://github.com/cloudflare/agent-skills-discovery-rfc), [agentskills.io](https://agentskills.io/).

#### `api-catalog`

- **Category:** `agentEndpoints`
- **Weight:** 0 (informational)
- **What it tests:** Whether `/.well-known/api-catalog` ([RFC 9727](https://www.rfc-editor.org/rfc/rfc9727)) returns a valid Linkset ([RFC 9264](https://www.rfc-editor.org/rfc/rfc9264)) pointing agents at OpenAPI specs and developer documentation.
- **Returns `pass`** when `GET /.well-known/api-catalog` (with `Accept: application/linkset+json, application/json`) returns a 2xx response containing parseable JSON whose root contains a `linkset` array, OR whose root itself is an array.
- **Returns `fail`** when non-2xx status, non-JSON body, or no linkset shape is present in the parsed JSON.
- **Returns `neutral`** when the endpoint is unreachable at the network level.

---

## 6. Status

The status enum is exactly three values, normative on the wire:

| value     | semantic                                                                                                                                                                                                                                                         |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pass`    | The check evaluated successfully and met its pass criteria. Contributes positively to score (numerator + denominator) when the check is scored.                                                                                                                  |
| `fail`    | The check evaluated successfully and did NOT meet its pass criteria. Contributes negatively to score (denominator only) when the check is scored.                                                                                                                |
| `neutral` | The check evaluated, but the result is informational only — the observation was inconclusive, the check is informational by design, or the check has deliberately opted out of the score denominator. EXCLUDED from both numerator and denominator of the score. |

### 6.1 Explicit non-goals (v1.0)

- **No `error` status.** v1.0 does not define a fourth status value for runtime failure modes. Implementations that experience a per-check runtime failure (parse error, internal exception, dependency unavailable) MUST map it to one of the three defined statuses:
  - Use **`fail`** when the check executed but encountered an internal error that prevented evaluation against pass criteria.
  - Use **`neutral`** when the observation could not be made (e.g., network failure, dependency missing, infrastructure unavailable) — i.e., when the implementation has no basis for asserting either pass or fail.
    Implementations MAY surface error details inside the per-check `details` payload, but no part of the wire status field reveals the error.
- **No `skipped` status.** Dependency-skipped checks (those whose hard-declared dependencies did not pass) MUST emit `neutral` with a clear `message` indicating the skip reason.
- **No `warn` status.** Tiered verdicts (e.g., `llms-txt-size` distinguishing pass/warn/fail) MUST be carried inside the per-check `details` payload, not on the status field.

v2.0+ MAY introduce additional status values; until then, the three-value enum is closed.

### 6.2 How status interacts with scoring

Status is the primary input to the scoring formula defined in [§7](#7-scoring). In summary:

- `pass` of a scored check (weight > 0) contributes its weight to BOTH numerator (passWeight) and denominator (passWeight + failWeight).
- `fail` of a scored check contributes its weight to denominator only.
- `neutral` of any check is excluded from both numerator and denominator.
- Informational checks (weight = 0) are excluded from both regardless of status — their statuses are emitted for consumer convenience and gate evaluation but do not alter the score.

The full formula, edge-case behavior, cluster math, and per-category scoring rules are normative and defined in [§7](#7-scoring).

---

## 7. Scoring

Scoring in v1.0 is fully deterministic: given identical check results and identical cluster verdicts, every conforming scanner produces the same score. Two scans of the same site at the same moment SHOULD produce identical scores (subject to per-scan probe sampling for the small number of checks that sample multiple URLs).

### 7.1 Top-level score formula

```
score = round(100 * passWeight / (passWeight + failWeight))
```

Definitions:

- **`passWeight`** — sum of `weight` values across all scored checks (`weight > 0`) whose `status === "pass"` after coefficient-cluster adjustment (see [§7.3](#73-cluster-math)).
- **`failWeight`** — sum of `weight` values across all scored checks whose `status === "fail"` after coefficient-cluster adjustment.
- **`score`** — integer in the range `[0, 100]`.

Exclusion rules:

1. Checks with `status === "neutral"` MUST be excluded from BOTH `passWeight` and `failWeight` (i.e., excluded from both numerator and denominator).
2. Checks with `weight === 0` (informational checks) MUST be excluded entirely from scoring regardless of status.
3. A check whose effective weight has been zeroed by a coefficient cluster (see [§7.3](#73-cluster-math)) MUST be excluded from BOTH numerator and denominator (its post-coefficient weight is `0`, so it contributes nothing on either side).

**Rounding:** half-up to nearest integer (e.g., `0.5` rounds to `1`, `49.5` rounds to `50`). Implementations MUST use this rounding rule. (Standard `round` behavior in most languages; specifying it removes ambiguity vs. banker's-rounding alternatives.)

**Divide-by-zero edge case (NORMATIVE):** When `passWeight + failWeight === 0` — i.e., every scored check returned `neutral`, or every scored check had its weight zeroed by a coefficient cluster — implementations MUST emit `score = 0`. Implementations MUST NOT emit `null`, an error, or any value other than `0` in this case.

### 7.2 Per-category score

Each entry in `categoryReports[]` MUST emit an integer `score` field computed by the same formula scoped to that category's checks only:

```
categoryScore = round(100 * categoryPassWeight / (categoryPassWeight + categoryFailWeight))
```

Where the weights are summed only across checks belonging to that category.

**Per-category scores are NOT subject to cluster math.** Coefficient adjustments to per-check weights and final-score caps apply ONLY to the top-level `score` field, not to per-category subscores. Per-category subscores describe category posture in isolation; cluster effects are global signals about overall site shape.

The same divide-by-zero rule applies: if a category has no scored checks contributing pass/fail weight, its `score` MUST be `0`.

### 7.3 Cluster math

v1.0 defines exactly **three** clusters, emitted in fixed order as the top-level `clusters[]` array:

1. `htmlPath` — kind `coefficient`
2. `spaRenderingCap` — kind `cap`
3. `noViablePathCap` — kind `cap`

Conforming scanners MUST emit all three cluster verdicts in this order, regardless of whether any are triggered.

#### 7.3.1 Cluster `htmlPath` (coefficient)

| Field         | Value                                                       |
| ------------- | ----------------------------------------------------------- |
| `name`        | `"htmlPath"`                                                |
| `kind`        | `"coefficient"`                                             |
| `appliesTo`   | `["page-size-html"]` (always present)                       |
| `coefficient` | `1` when not triggered; `0` when triggered. Always present. |

- **Trigger condition:** `rendering-strategy.status === "fail"`. (Note: the trigger is specifically `fail`, NOT `!= "pass"`. A `neutral` rendering verdict — for example, when headless rendering is unavailable — does NOT trigger this cluster.)
- **Effect when triggered:** before scoring, multiply the effective weight of each check listed in `appliesTo` by `coefficient`. With `coefficient: 0`, `page-size-html`'s effective weight becomes `0` and the check is excluded from both `passWeight` and `failWeight`. Coefficient effects are pre-aggregation, applied to per-check weights before the score formula runs.
- **Effect when not triggered:** `coefficient: 1` is a no-op; weights are unchanged.

Triggered shape (illustrative):

```json
{
  "name": "htmlPath",
  "kind": "coefficient",
  "triggered": true,
  "coefficient": 0,
  "appliesTo": ["page-size-html"],
  "message": "..."
}
```

Untriggered shape (illustrative):

```json
{
  "name": "htmlPath",
  "kind": "coefficient",
  "triggered": false,
  "coefficient": 1,
  "appliesTo": ["page-size-html"],
  "message": "..."
}
```

#### 7.3.2 Cluster `spaRenderingCap` (cap)

| Field      | Value                                                                                       |
| ---------- | ------------------------------------------------------------------------------------------- |
| `name`     | `"spaRenderingCap"`                                                                         |
| `kind`     | `"cap"`                                                                                     |
| `capScore` | Present ONLY when `triggered === true`. Permitted values: `39` (severe) or `59` (moderate). |

- **Trigger condition:** `rendering-strategy.status === "fail"`. (Same trigger as `htmlPath`; the two clusters fire together when rendering fails.)
- **Severity selection (NORMATIVE):**
  - `capScore: 39` (severe) when the rendering-strategy fail was caused by a SPA-shell short-circuit OR the rendered/plain content-length ratio is `>= 2.0`.
  - `capScore: 59` (moderate) otherwise (i.e., the ratio is in `(1.20, 2.0)` after rendering-strategy classified the page as fail).
- **Effect when triggered:** the FINAL top-level `score` is clamped to `min(score, capScore)`.
- **When not triggered:** the `capScore` field MUST be omitted from the cluster object.

Triggered shape (illustrative):

```json
{
  "name": "spaRenderingCap",
  "kind": "cap",
  "triggered": true,
  "capScore": 39,
  "message": "..."
}
```

Untriggered shape (illustrative):

```json
{
  "name": "spaRenderingCap",
  "kind": "cap",
  "triggered": false,
  "message": "..."
}
```

#### 7.3.3 Cluster `noViablePathCap` (cap)

| Field      | Value                                                          |
| ---------- | -------------------------------------------------------------- |
| `name`     | `"noViablePathCap"`                                            |
| `kind`     | `"cap"`                                                        |
| `capScore` | Present ONLY when `triggered === true`. Permitted value: `39`. |

- **Trigger condition (NORMATIVE):** ALL FOUR of the following checks have `status !== "pass"` AND at least TWO of them have `status === "fail"` (i.e., explicit fail, not merely neutral):
  - `llms-txt-exists`
  - `rendering-strategy`
  - `markdown-url-support`
  - `markdown-negotiation`
- **Effect when triggered:** the FINAL top-level `score` is clamped to `min(score, 39)`.
- **Rationale for the "≥ 2 explicit fails" threshold:** several of these inputs can return `neutral` for benign reasons (probe-tooling unavailability, network failure, sample-size insufficiency). Requiring at least two `fail` outcomes guards against false-positive triggering when a scanner cannot make reliable observations.
- **When not triggered:** the `capScore` field MUST be omitted from the cluster object.

#### 7.3.4 Composition order

The composition order of cluster effects on the top-level `score` is normative:

1. **Apply coefficient clusters** to per-check weights (mutate the effective weight pool).
2. **Compute base score** per the formula in [§7.1](#71-top-level-score-formula) using adjusted weights.
3. **Apply cap clusters** as `score = min(baseScore, lowestTriggeredCap)`.

Implementations that compute caps before coefficients, or interleave the operations differently, will produce non-conforming scores.

#### 7.3.5 Cap stacking

When multiple cap clusters trigger simultaneously, the LOWEST `capScore` wins. **Caps do NOT compound** (they do not multiply, do not subtract). If both `spaRenderingCap` (cap `39` or `59`) and `noViablePathCap` (cap `39`) trigger, the final score is `min(baseScore, 39)`.

#### 7.3.6 Per-category scoring is exempt from cluster math

Per-category `score` values in `categoryReports[].score` MUST NOT have coefficients applied to them and MUST NOT be capped. Cluster math affects only the top-level `score` field.

### 7.4 Algorithm summary

A conforming implementation computes the top-level `score` as follows:

1. Run all 25 checks; collect results with `(id, category, status, weight)`.
2. Compute the three cluster verdicts per [§7.3.1](#731-cluster-htmlpath-coefficient) – [§7.3.3](#733-cluster-noviablepathcap-cap).
3. For each check whose `id` appears in any coefficient cluster's `appliesTo`, multiply its effective weight by that cluster's `coefficient`.
4. Compute `baseScore` per [§7.1](#71-top-level-score-formula) using adjusted weights.
5. If any cap cluster has `triggered === true`, set `finalScore = min(baseScore, lowestTriggeredCapScore)`. Otherwise `finalScore = baseScore`.
6. For each category, compute `categoryReports[].score` per [§7.2](#72-per-category-score) with the ORIGINAL (un-adjusted) weights, scoped to that category's checks. NO cluster math applies here.
7. Emit `score = finalScore` at the top level and per-category subscores in `categoryReports[].score`.

A complete worked example with concrete numbers appears in [Annex B](#annex-b--worked-example).

---

## 8. Levels

### 8.1 Three levels

v1.0 defines exactly three levels. Conforming scanners MUST NOT emit `level` values outside this enum.

| `level` (integer) | `levelName` (string)   |
| ----------------- | ---------------------- |
| `1`               | `"Basic Web Presence"` |
| `2`               | `"AI-Aware"`           |
| `3`               | `"Agent-Optimized"`    |

`level` is an INTEGER on the wire (see [§10.1](#101-top-level-shape-13-fields)). It MUST NOT be emitted as a string.

### 8.2 Gate logic

Level is **gate-driven**, not score-driven. Each level transition is governed by a single named check whose `status` MUST equal `"pass"` for the transition to occur.

| Transition | Gate check             |
| ---------- | ---------------------- |
| L1 → L2    | `content-signals`      |
| L2 → L3    | `markdown-negotiation` |

- Every site starts at **Level 1** by default. There is no L1 entry gate.
- A site advances to L2 if and only if `content-signals.status === "pass"`.
- A site advances to L3 if and only if `markdown-negotiation.status === "pass"`. (L3 also requires that the L1→L2 gate is met, by transitivity: the level system is monotonic; L3 sites are also L2 sites.)
- Gate semantics are **AND** across `requirements` if a future spec version introduces multi-check gates. v1.0 uses single-check gates throughout.

### 8.3 `nextLevel` emission

When emitted, `nextLevel` is an object describing the target of the next available level transition.

Object shape:

```json
{
  "level": <integer>,
  "levelName": <string>,
  "requirements": [<check-id>, ...]
}
```

- `level` — INTEGER, enum `[2, 3]`. The level the site would advance TO. (Sites at L1 advance TO L2; sites at L2 advance TO L3. There is no `level: 1` here because there is nothing prior to advance from.)
- `levelName` — STRING, enum `["AI-Aware", "Agent-Optimized"]`, corresponding 1:1 to `level`.
- `requirements` — non-empty ARRAY of kebab-case check IDs whose `status` MUST flip to `"pass"` for the site to advance. v1.0 always emits a single-element array per transition (single-check gates).

**Conformance rule:** When the site is at the highest defined level (v1.0: `level === 3`), conforming scanners MUST **OMIT** the `nextLevel` field entirely. Scanners MUST NOT emit `nextLevel: null`, MUST NOT emit `nextLevel: {}`, and MUST NOT emit a stub or sentinel value.

The schema reflects this requirement: `nextLevel` appears in `properties` (so its shape is constrained when present) but NOT in the top-level `required` array (so its absence is valid).

### 8.4 Score and level are independent

Score and level are **decoupled observable signals**. The two carry distinct, complementary information:

- A site MAY have `score === 0` and still reach `level === 2` if `content-signals.status === "pass"` (an informational, weight-`0` check whose pass status is the L1→L2 gate). Concretely: a site that publishes only `content-signals` and fails every scored check would score `0` at level `2`.
- A site MAY have `score === 99` and still remain at `level === 1` if `content-signals.status !== "pass"`.

Consumers of scan output SHOULD report both `score` and `level` together; treating either as a proxy for the other distorts the spec's intent.

---

## 9. Profiles

### 9.1 Profile concept

A **Profile** is a JSON document declaring which checks are active and their per-check weights for a given vertical, use case, or scanner deployment. A profile binds the abstract Specification to a concrete measurement workload.

v1.0 ships **exactly one** profile: `b2b-saas`. v1.0 does NOT define a custom-profile mechanism; all v1.0-conforming scanners MUST emit `profile: "b2b-saas"` on the wire.

### 9.2 The `b2b-saas` profile

| Field                 | Value                      |
| --------------------- | -------------------------- |
| `profile_id`          | `"b2b-saas"`               |
| `profile_name`        | `"B2B SaaS"`               |
| `spec_version`        | `"1.0.0"`                  |
| Active checks         | 25 (the full v1.0 catalog) |
| Sum of scored weights | 80                         |

**Description:** Default profile for v1.0 of the Agent-Adoption Specification. Applies to commercial software-as-a-service businesses targeting business customers. All 25 spec checks are applicable; the weight assigned to each check determines its contribution to the score, and checks with `weight: 0` are informational.

Per-check weights in `b2b-saas` are normative for this profile and MUST match the catalog in [§5](#5-checks-catalog).

### 9.3 Profile JSON shape (illustrative for v1.0)

The structure of profile JSON is **illustrative and NOT normatively schema-validated** in v1.0 (a formal `profile-v1.schema.json` is deferred to v1.1+ or v2.0).

Top-level fields:

| Field          | Type                       | Meaning                                                      |
| -------------- | -------------------------- | ------------------------------------------------------------ |
| `profile_id`   | string                     | Stable identifier (matches the `profile` field on the wire). |
| `profile_name` | string                     | Human-readable label.                                        |
| `spec_version` | string                     | Semver of the Specification this profile binds to.           |
| `description`  | string                     | Free-text description of the profile's intended scope.       |
| `checks`       | object (keyed by check-id) | Per-check weight and category assignment.                    |

`checks` is a **keyed object**, not an array, for direct lookup by check ID:

```json
"checks": {
  "robots-txt-exists": { "category": "discoverability", "weight": 7 },
  "sitemap-exists":    { "category": "discoverability", "weight": 4 }
}
```

Each per-check entry has:

| Sub-field  | Type    | Meaning                                                                                                                   |
| ---------- | ------- | ------------------------------------------------------------------------------------------------------------------------- |
| `category` | string  | One of the four category enum values (camelCase). MUST match the canonical category for that check ([§4](#4-categories)). |
| `weight`   | integer | One of `{0, 4, 5, 6, 7, 8, 10}`. Determines the check's contribution to scoring.                                          |

**Casing convention notice:** Profile JSON uses snake_case for top-level field names (`profile_id`, `profile_name`, `spec_version`). This is INTENTIONAL and distinct from the camelCase wire output; profile JSON is human-authored configuration metadata, while wire output is a machine-emitted scan result. The two artifacts use different conventions.

### 9.4 v1.0 profile conformance

- A v1.0-conforming scanner MUST emit `profile: "b2b-saas"` as a top-level wire field.
- A v1.0-conforming scanner MUST apply the per-check weights as defined in the `b2b-saas` profile (which match the catalog in [§5](#5-checks-catalog)).
- A v1.0-conforming scanner MUST NOT advertise a custom or vendor-specific profile identifier in the `profile` field for v1.0 conformance. (Custom profiles are deferred to a future minor or major version.)

---

## 10. Output format

The wire format is JSON. Conforming scanners emit a single self-contained object per scan. All object keys and category enum values use **camelCase**; check IDs use **kebab-case**; timestamps use **ISO 8601 with `Z` suffix**; `level`, `score`, `weight`, all duration fields, and all category counts are **integers**.

### 10.1 Top-level shape (13 fields)

All 13 fields are emitted on every scan response in normal mode, with one exception: `nextLevel` is omitted at L3 ([§8.3](#83-nextlevel-emission)). Conforming scanners MUST emit each REQUIRED field on every scan and MUST NOT emit any other top-level fields (the schema is closed at the root via `additionalProperties: false`).

| Field             | Type                        | Required    | Description                                                                                                                                                                                                                                  |
| ----------------- | --------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `specVersion`     | string (semver)             | yes         | The Specification version this scan conforms to (e.g., `"1.0.0"`). Pattern `^\d+\.\d+\.\d+$`.                                                                                                                                                |
| `profile`         | string                      | yes         | Identifier of the profile applied (v1.0: always `"b2b-saas"`).                                                                                                                                                                               |
| `scanner`         | object                      | yes         | `{ name: string, version: string }`. The scanner's own name and version (implementation-chosen).                                                                                                                                             |
| `domain`          | string                      | yes         | The brand root domain (eTLD+1) input to the scan. Lowercase, no scheme, no trailing slash.                                                                                                                                                   |
| `finalUrl`        | string (URI)                | yes         | Fully-qualified URL the scanner ended on after following redirects.                                                                                                                                                                          |
| `scannedAt`       | string (ISO 8601 date-time) | yes         | UTC timestamp at which the scan completed. MUST end with `Z`.                                                                                                                                                                                |
| `score`           | integer `[0, 100]`          | yes         | The aggregate score per [§7](#7-scoring).                                                                                                                                                                                                    |
| `level`           | integer (enum `[1, 2, 3]`)  | yes         | The level per [§8](#8-levels). MUST be an integer.                                                                                                                                                                                           |
| `levelName`       | string (enum)               | yes         | One of `"Basic Web Presence"`, `"AI-Aware"`, `"Agent-Optimized"`, corresponding to `level`.                                                                                                                                                  |
| `nextLevel`       | object                      | conditional | Present at L1 and L2; OMITTED at L3 (see [§8.3](#83-nextlevel-emission)).                                                                                                                                                                    |
| `categoryReports` | array, length exactly 4     | yes         | Per-category reports in canonical order: `discoverability`, `accessControl`, `contentReadability`, `agentEndpoints`. The schema enforces this order via `prefixItems` with a fixed `category` const per position.                            |
| `meta`            | object                      | yes         | Telemetry and counters (see [§10.5](#105-meta-object-shape)).                                                                                                                                                                                |
| `clusters`        | array, length exactly 3     | yes         | Cluster verdicts in canonical order: `htmlPath`, `spaRenderingCap`, `noViablePathCap`. Top-level field because cluster math affects the published score and consumers need it to reproduce or audit the score (see [§7.3](#73-cluster-math) and [§10.6](#106-clusters-array)).                                                                                                                            |

A complete example appears in [Annex B](#annex-b--worked-example).

### 10.2 Per-check object shape (10 required fields + 1 conditional)

Every check appears inside `categoryReports[].checks[]`. The 10 required fields MUST be present on every check; `details` is conditional (see below).

| Field         | Type                                    | Required    | Description                                                                                                                                                                         |
| ------------- | --------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`          | string (kebab-case)                     | yes         | The canonical check ID. Pattern `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`. v1.0 defines 25 IDs ([§5](#5-checks-catalog)).                                                                     |
| `category`    | string (enum)                           | yes         | One of the four category enum values. MUST equal the enclosing `categoryReport.category`.                                                                                           |
| `status`      | string (enum)                           | yes         | Exactly one of `"pass"`, `"fail"`, `"neutral"` ([§6](#6-status)).                                                                                                                   |
| `scored`      | boolean                                 | yes         | `true` iff `weight > 0`. Equivalent to `(weight > 0)`; emitted for client convenience.                                                                                              |
| `weight`      | integer (enum `{0, 4, 5, 6, 7, 8, 10}`) | yes         | Per-check weight from the v1.0 catalog.                                                                                                                                             |
| `message`     | string                                  | yes         | One-line human-readable verdict explaining the status.                                                                                                                              |
| `description` | string                                  | yes         | One-paragraph description of what the check evaluates (constant per check).                                                                                                         |
| `durationMs`  | integer (≥ 0)                           | yes         | Wall-clock execution time of this check in milliseconds.                                                                                                                            |
| `specUrls`    | array of URI strings                    | yes         | URLs of relevant external standards/RFCs. MAY be empty for checks with no published standard.                                                                                       |
| `dependsOn`   | array of check-id strings               | yes         | Check IDs this check depends on. Empty array if independent.                                                                                                                        |
| `details`     | object                                  | conditional | Check-specific structured payload. **Present** when the check produces a structured payload (typical happy-path execution). **Absent** on dependency-skip and runtime-error paths, where the verdict is fully carried by `status` + `message`. Schema is per-check; consumers MUST parse defensively (e.g., `check.details?.kind`). See [§11.2](#112-may-allowances) and [§11.3](#113-must-not-requirements) for constraints when the field IS present. |

Notes:

- The `details` object, when present, MAY contain a `kind` field whose value SHOULD equal the enclosing check's `id` (RECOMMENDED for consumer-side discrimination). This is not required by the schema.
- `details.body` is FORBIDDEN. Conforming scanners MUST NOT include a `body` key inside `details` (see [§11.3](#113-must-not-requirements)). Raw HTTP response bodies do not belong on the wire.
- Consumers SHOULD treat `details` as optional and access nested fields via optional chaining or equivalent guarded reads.

### 10.3 Per-`categoryReport` object shape (7 fields)

Each entry in the top-level `categoryReports` array conforms to:

| Field         | Type                   | Required | Description                                                                                |
| ------------- | ---------------------- | -------- | ------------------------------------------------------------------------------------------ |
| `category`    | string (enum)          | yes      | One of `"discoverability"`, `"accessControl"`, `"contentReadability"`, `"agentEndpoints"`. |
| `description` | string                 | yes      | Short description of what the category measures (constant per category).                   |
| `score`       | integer `[0, 100]`     | yes      | Per-category subscore per [§7.2](#72-per-category-score).                                  |
| `passed`      | integer (≥ 0)          | yes      | Count of checks in this category with `status === "pass"` (includes informational checks). |
| `failed`      | integer (≥ 0)          | yes      | Count of checks with `status === "fail"`.                                                  |
| `neutral`     | integer (≥ 0)          | yes      | Count of checks with `status === "neutral"`.                                               |
| `checks`      | array of check objects | yes      | All checks in this category, in scanner-defined order; MUST be non-empty.                  |

The four `categoryReports` entries MUST appear in the canonical order: `discoverability`, then `accessControl`, then `contentReadability`, then `agentEndpoints`. The schema enforces this via `prefixItems` with a fixed `category` const per position.

### 10.4 `nextLevel` object shape

Defined in [§8.3](#83-nextlevel-emission). When present:

| Field          | Type                                            | Required | Description                                           |
| -------------- | ----------------------------------------------- | -------- | ----------------------------------------------------- |
| `level`        | integer (enum `[2, 3]`)                         | yes      | Level the site would advance TO.                      |
| `levelName`    | string (enum `["AI-Aware", "Agent-Optimized"]`) | yes      | Human-readable label of the next level.               |
| `requirements` | non-empty array of check-id strings             | yes      | Check IDs that must flip to `"pass"` for advancement. |

OMITTED entirely at L3 (NOT nullified, NOT empty-objected).

### 10.5 `meta` object shape

Always present at top level. 4 fields:

| Field             | Type          | Required | Description                                                                                           |
| ----------------- | ------------- | -------- | ----------------------------------------------------------------------------------------------------- |
| `checksEvaluated` | integer (≥ 0) | yes      | Number of checks the scanner evaluated. v1.0 expects `25`.                                            |
| `checksSkipped`   | integer (≥ 0) | yes      | Number of checks the scanner did not evaluate (e.g., due to profile filtering or runtime conditions). |
| `scanDurationMs`  | integer (≥ 0) | yes      | Total wall-clock duration of the scan, server-side, in milliseconds.                                  |
| `counters`        | object        | yes      | Implementation-defined telemetry counters (see below).                                                |

**`counters` shape.** The value is a nested object: `{<counterGroup>: {<metricName>: integer}}`. Both levels of keys are open (`additionalProperties: true`) so implementations can add or rename groups without a schema break, provided each leaf value remains an integer ≥ 0.

Example (illustrative — actual group names and metric names are implementation-defined):

```json
"counters": {
  "fetches": { "total": 47, "success": 35, "wafBlocked": 0, "notFound": 8, "failed": 4 }
}
```

The schema does NOT enumerate counter group names or metric names. Consumers MUST treat `counters` as opaque for conformance purposes.

### 10.6 `clusters` array

A top-level array of length exactly 3. Cluster verdicts appear in this canonical order: `htmlPath`, `spaRenderingCap`, `noViablePathCap`. The schema enforces both the length and the per-position `name` const via `prefixItems`. Conforming scanners MUST emit all three cluster verdicts on every scan, including non-triggered ones.

**Why top-level:** the cluster math is part of the normative score-computation discipline (see [§7.3](#73-cluster-math)). Coefficient clusters multiply per-check weights; cap clusters clamp the final score. A consumer cannot reproduce or audit the published `score` without the cluster verdicts. Anything that influences scoring methodology is core spec, not vendor-extension data — so the wire surface is at the root, not buried in a vendor namespace.

Each cluster verdict object:

| Field         | Type                      | Required    | Notes                                                                                                             |
| ------------- | ------------------------- | ----------- | ----------------------------------------------------------------------------------------------------------------- |
| `name`        | string (enum)             | yes         | One of `"htmlPath"`, `"spaRenderingCap"`, `"noViablePathCap"`.                                                    |
| `kind`        | string (enum)             | yes         | `"coefficient"` or `"cap"`.                                                                                       |
| `triggered`   | boolean                   | yes         | Whether the cluster's effect was applied.                                                                         |
| `coefficient` | integer (enum `[0, 1]`)   | conditional | Present only on `kind: "coefficient"` clusters. Always emitted for those.                                         |
| `appliesTo`   | array of check-id strings | conditional | Present only on `kind: "coefficient"` clusters. Lists the checks whose weights are scaled.                        |
| `capScore`    | integer                   | conditional | Present only on `kind: "cap"` clusters AND when `triggered === true`. MUST be omitted when `triggered === false`. |
| `message`     | string                    | yes         | Human-readable explanation of the cluster's verdict.                                                              |

Per-cluster trigger conditions and cap-value enums: see [§7.3](#73-cluster-math).

### 10.7 Vendor extension surfaces

v1.0 does NOT define a top-level vendor namespace. The wire surface is closed at the root — the schema declares `additionalProperties: false` on the root object, so implementations MUST NOT emit unrecognized top-level fields.

Implementations that need to surface vendor-specific data have two open surfaces:

- **Per-check `details` object** — the schema declares `additionalProperties: true` on `details` so implementations MAY emit any per-check vendor fields. The single carve-out: `details.body` is FORBIDDEN regardless of mode (see [§10.2](#102-per-check-object-shape-11-fields) and [§11.3](#113-must-not-requirements)). Use `details` for per-check evidence, debug payloads, or vendor-specific structured data tied to one check.
- **`meta.counters` object** — open at both levels (`additionalProperties: true` on `counters` and on each counter group). Use this for telemetry (fetch counts, retry counts, internal subprocess metrics, etc.) per [§10.5](#105-meta-object-shape).

Consumers MUST NOT reject scan output that contains unknown keys inside `details` or `meta.counters`, provided the rest of the document validates against the schema.

A future minor version MAY introduce a top-level vendor namespace if a real cross-implementation use case emerges; v1.0 deliberately omits it to keep the wire surface minimal.

---

## 11. Conformance

### 11.1 MUST requirements

A conforming v1.0 scanner MUST:

1. Emit all 13 top-level fields per [§10.1](#101-top-level-shape-13-fields) (with `nextLevel` omitted only when `level === 3`).
2. Emit all 25 checks listed in the [§5](#5-checks-catalog) catalog under their declared categories with their declared weights.
3. Use the `status` enum exactly as defined in [§6](#6-status): only `"pass"`, `"fail"`, `"neutral"`.
4. Use `level` as an INTEGER (`1`, `2`, or `3`) and emit `levelName` as the corresponding string from the enum in [§8.1](#81-three-levels).
5. Apply cluster math and the score formula exactly as specified in [§7](#7-scoring), including the divide-by-zero rule (`score = 0` when no scored checks contribute) and the composition order (coefficients → base score → caps).
6. Emit a top-level `clusters` array with all three cluster verdicts in canonical order, including non-triggered ones.
7. Use camelCase for all object keys and category enum values on the wire.
8. Use INTEGER types for `level`, `score`, `weight`, all `*DurationMs` fields, and all category counts (`passed`, `failed`, `neutral`, `checksEvaluated`, `checksSkipped`, all `meta.counters` leaf values).
9. Use ISO 8601 timestamps with `Z` suffix for `scannedAt`.
10. Emit `specVersion` as a dotted-triple semver string identifying the version of the Specification the scanner conforms to (e.g., `"1.0.0"`).
11. Emit `profile: "b2b-saas"` as the only valid profile identifier in v1.0.

### 11.2 MAY allowances

A conforming v1.0 scanner MAY:

1. Emit additional fields inside any per-check `details` object (`additionalProperties: true`), EXCEPT for `details.body` which is FORBIDDEN.
2. Emit additional `meta.counters.*` groups and metrics, provided the nested-object shape and integer-leaf constraint of [§10.5](#105-meta-object-shape) are preserved.
3. Emit additional URLs in any check's `specUrls` array beyond those listed in the [§5](#5-checks-catalog) catalog.
4. Choose its own request headers, retry policy, and timeout values for HTTP probes, unless explicitly constrained in a per-check definition.
5. Choose its own URL sampling strategy within the sample-size bounds declared by per-check definitions in [§5](#5-checks-catalog) (e.g., the choice of which 5 sitemap URLs to probe for `markdown-url-support` is implementation-defined; the requirement that at least 3 measurable probes be obtained is normative).
6. Set the `details.kind` field equal to the enclosing check's `id` (RECOMMENDED for consumer discrimination but not required).
7. Emit any scanner `name` and `version` strings in the `scanner` object; these are implementation-defined.

### 11.3 MUST NOT requirements

A conforming v1.0 scanner MUST NOT:

1. Use snake_case for top-level wire field names.
2. Emit `level` as a string (it is always an integer; `levelName` is the separate string form).
3. Use any `status` value other than `"pass"`, `"fail"`, or `"neutral"`. Specifically, `"error"` and `"skipped"` are FORBIDDEN.
4. Use any `weight` value outside the closed set `{0, 4, 5, 6, 7, 8, 10}`.
5. Add or remove checks from the 25-check registry without a `specVersion` bump (additions require at minimum a MINOR bump; removals or breaking semantic changes require a MAJOR bump — see [§12](#12-versioning)).
6. Emit `details.body`. The schema explicitly forbids this property under per-check `details`.
7. Emit `nextLevel: null` or `nextLevel: {}` at level 3. The field MUST be OMITTED entirely.
8. Emit `clusters` array entries out of canonical order, or with fewer/more than 3 entries.
9. Emit `categoryReports` entries out of canonical order, or with fewer/more than 4 entries.

### 11.4 Optional debug-mode considerations (informative)

Implementations MAY support debug or diagnostic output modes that add non-normative fields to per-check objects (for example, raw evidence arrays, per-fetch audit fields, or fix-suggestion payloads). These modes are NOT specified in v1.0; implementations choose their own surface area.

For conformance purposes, the v1.0 spec is judged against **normal-mode** output only. Debug-mode output is OUT OF SCOPE for conformance testing. v2.0+ MAY standardize a debug-mode surface once implementer experience accumulates.

---

## 12. Versioning

### 12.1 Semver discipline

The Specification follows semantic versioning (`MAJOR.MINOR.PATCH`):

- **MAJOR (`vN.0.0`):** Breaking wire-format changes. Examples: removing checks from the registry, changing the meaning of an existing enum value, changing the structure or semantics of a top-level field, dropping a `MUST`-level requirement.
- **MINOR (`v1.x.0`):** Non-breaking additions. Examples: adding new checks, adding new clusters, introducing a new optional top-level field, introducing a vendor-extension namespace if implementer demand emerges.
- **PATCH (`v1.0.x`):** Editorial fixes and clarifications without behavior change. Examples: typo corrections, tightening ambiguous phrasing, recalibrating threshold constants when the new constant is the empirically-correct interpretation of what was already meant.

### 12.2 Tag immutability

Each tagged release of the Specification is **permanent** and accessible at its version-suffixed canonical URL. v1.0 will remain accessible indefinitely, even after v2.0 (or later) is published.

A conforming implementation declares which version it implements via the `specVersion` field on the wire. An implementation that conforms to v1.0 will continue to be conformant against the v1.0 spec indefinitely; later spec versions do not retroactively invalidate earlier conforming implementations.

### 12.3 Erratum mechanism

Editorial bugs in published Specification text — typos, ambiguous phrasing, broken links, missing clarifications — MAY be corrected via a PATCH release (e.g., v1.0.1, v1.0.2).

A patched release MUST NOT change normative behavior. It MAY:

- Fix a typo.
- Tighten an ambiguous sentence to match the originally-intended interpretation.
- Add a clarifying example or note.
- Correct a calibration constant when the new value is demonstrably the value the spec already meant (e.g., a number that was wrong in the prose but correct in the schema).

A patched release MUST NOT:

- Change the trigger conditions of any check.
- Change the value or interpretation of any enum.
- Add or remove fields from the wire format.
- Change the score formula or composition order.

Behavioral changes belong in a MINOR (additive) or MAJOR (breaking) release.

### 12.4 The `specVersion` field

Conforming scanners MUST emit `specVersion` as the dotted-triple semver string identifying the Specification version they conform to. Examples: `"1.0.0"`, `"1.0.1"`, `"1.1.0"`. The pattern is `^\d+\.\d+\.\d+$` (no pre-release suffixes, no build metadata).

A scanner that conforms to v1.0 emits `"1.0.0"`. A scanner that conforms to a future patched v1.0.1 emits `"1.0.1"`. A scanner that conforms to a future v1.1.0 (with additional checks) emits `"1.1.0"`. Implementations MUST NOT emit a `specVersion` they do not actually conform to.

---

## Annex A — Defensive framing

### A.1 What the Specification measures (positive framing)

The Specification measures **observable site state**: the presence and well-formedness of standardized agent-readiness signals an automated HTTP client would expect to find. Specifically:

- All 25 checks are determined deterministically from HTTP-level probes (and, in the case of `rendering-strategy`, an optional headless render).
- All status verdicts (`pass`, `fail`, `neutral`), all scores, and all level assignments are direct consequences of the observed signals.
- Two scans of the same site at the same moment, run by conforming scanners with comparable network access, SHOULD produce identical scores, levels, and per-check verdicts (modulo per-scan probe sampling for the small number of checks that sample multiple URLs).

The output is a **measurement contract**: it answers the question "are the standardized agent-readiness signals an agent would look for present and well-formed on this site, right now?"

### A.2 What the Specification does NOT measure (anti-claims)

The Specification deliberately does NOT measure, and conforming output MUST NOT be interpreted as a proxy for, any of the following:

- Predicted volume of AI traffic the site will receive.
- Conversion or revenue impact of agent-readiness investments.
- Frequency or quality of brand mentions in agent outputs (LLM citations, agent recommendations, etc.).
- Search-result rank in any search engine, crawler, or retrieval system.
- Whether any specific agent (existing or future) will visit, cite, parse, or correctly interpret the site.
- "Future-proofing" against unknown future agent behaviors or unspecified protocols.

A high score does NOT guarantee agent visibility. A low score does NOT predict agent invisibility (agents tolerate missing signals via heuristics).

### A.3 Recommended consumption rules

Consumers of scan output SHOULD:

1. Use the score and level as ONE input to agent-readiness decisions, not the SOLE input.
2. Combine scan output with site-specific business context (audience, content strategy, regulatory posture, competitive landscape).
3. Watch for spurious correlations between score and downstream metrics; the Specification is descriptive and makes no causal claim.
4. NOT chase the score at the expense of legitimate site design, accessibility, performance, or user experience.
5. Treat per-category scores and individual check verdicts as more actionable signals than the aggregate `score`; the aggregate is a summary, not an instruction.

---

## Annex B — Worked example

This annex walks a hypothetical scan against `example.com` end-to-end, demonstrating the score formula, gate logic, and cluster verdicts. The numbers are self-consistent: the score, level, and per-category subscores in the final JSON each equal what the formulas in [§7](#7-scoring) and [§8](#8-levels) yield from the per-check inputs.

The example uses no specific implementation; the `scanner` object identifies a generic `"Example Conforming Scanner"`. Per-check `details` payloads are minimized to a single `kind` field for brevity (real-world scans typically emit richer `details`).

### B.1 Per-check status assignment

The fictional `example.com` scan classifies each of the 25 checks as follows. (Order: canonical category, then weight descending, then ID alphabetic.)

| id                              | category           | weight | scored | status   | contributes to                  |
| ------------------------------- | ------------------ | ------ | ------ | -------- | ------------------------------- |
| `robots-txt-exists`             | discoverability    | 7      | true   | **pass** | passWeight (+7)                 |
| `sitemap-exists`                | discoverability    | 4      | true   | **fail** | failWeight (+4)                 |
| `link-headers`                  | discoverability    | 0      | false  | neutral  | —                               |
| `ai-bot-rules`                  | accessControl      | 7      | true   | **pass** | passWeight (+7)                 |
| `content-signals`               | accessControl      | 0      | false  | **pass** | — (gate: L1→L2 ✓)               |
| `web-bot-auth`                  | accessControl      | 0      | false  | neutral  | —                               |
| `robots-allow-all`              | accessControl      | 0      | false  | pass     | —                               |
| `rendering-strategy`            | contentReadability | 10     | true   | **pass** | passWeight (+10)                |
| `markdown-url-support`          | contentReadability | 8      | true   | neutral  | — (excluded)                    |
| `http-status-codes`             | contentReadability | 6      | true   | **pass** | passWeight (+6)                 |
| `page-size-html`                | contentReadability | 6      | true   | neutral  | — (excluded)                    |
| `markdown-negotiation`          | contentReadability | 5      | true   | **fail** | failWeight (+5) — gate: L2→L3 ✗ |
| `redirect-behavior`             | contentReadability | 4      | true   | **fail** | failWeight (+4)                 |
| `agents-md-detection`           | contentReadability | 0      | false  | fail     | —                               |
| `cache-header-hygiene`          | contentReadability | 0      | false  | pass     | —                               |
| `llms-txt-exists`               | contentReadability | 0      | false  | neutral  | —                               |
| `llms-txt-has-optional-section` | contentReadability | 0      | false  | neutral  | —                               |
| `llms-txt-size`                 | contentReadability | 0      | false  | neutral  | —                               |
| `llms-txt-valid`                | contentReadability | 0      | false  | neutral  | —                               |
| `mcp-server-card`               | agentEndpoints     | 10     | true   | neutral  | — (excluded)                    |
| `oauth-protected-resource`      | agentEndpoints     | 7      | true   | **fail** | failWeight (+7)                 |
| `oauth-discovery`               | agentEndpoints     | 6      | true   | neutral  | — (clean-404 hedge; excluded)   |
| `api-catalog`                   | agentEndpoints     | 0      | false  | fail     | —                               |
| `a2a-agent-card`                | agentEndpoints     | 0      | false  | fail     | —                               |
| `agent-skills`                  | agentEndpoints     | 0      | false  | fail     | —                               |

**Bookkeeping:** 4 scored pass (sum 30); 4 scored fail (sum 20); 4 scored neutral (sum 30, excluded); 13 informational (excluded). 12 + 13 = 25 ✓; sum of all scored weights = 80 ✓.

### B.2 Score arithmetic

#### Top-level score

```
passWeight = 7 + 7 + 10 + 6 = 30
failWeight = 4 + 5 + 4 + 7  = 20

weightedTotal = 30 + 20 = 50

score = round(100 * 30 / 50) = round(60.0) = 60
```

#### Per-category subscores

Each computed via the same formula scoped to that category's checks (no cluster math applies):

| category             | passW       | failW     | category score                  |
| -------------------- | ----------- | --------- | ------------------------------- |
| `discoverability`    | 7           | 4         | `round(100 * 7 / 11)` = **64**  |
| `accessControl`      | 7           | 0         | `round(100 * 7 / 7)` = **100**  |
| `contentReadability` | 10 + 6 = 16 | 5 + 4 = 9 | `round(100 * 16 / 25)` = **64** |
| `agentEndpoints`     | 0           | 7         | `round(100 * 0 / 7)` = **0**    |

#### Per-category counts

| category             | passed | failed | neutral | total    |
| -------------------- | ------ | ------ | ------- | -------- |
| `discoverability`    | 1      | 1      | 1       | 3        |
| `accessControl`      | 3      | 0      | 1       | 4        |
| `contentReadability` | 3      | 3      | 6       | 12       |
| `agentEndpoints`     | 0      | 4      | 2       | 6        |
| **totals**           | **7**  | **8**  | **10**  | **25** ✓ |

### B.3 Level calculation

| Transition | Gate check             | Required status | Met?                |
| ---------- | ---------------------- | --------------- | ------------------- |
| L1 → L2    | `content-signals`      | `pass`          | ✓ (assigned `pass`) |
| L2 → L3    | `markdown-negotiation` | `pass`          | ✗ (assigned `fail`) |

Therefore `level = 2`, `levelName = "AI-Aware"`, and `nextLevel` is emitted with the L3 advancement requirement:

```json
"nextLevel": {
  "level": 3,
  "levelName": "Agent-Optimized",
  "requirements": ["markdown-negotiation"]
}
```

### B.4 Cluster verdicts

All three clusters are NOT triggered:

- `htmlPath` does not trigger because `rendering-strategy.status === "pass"` (its trigger is `=== "fail"`). Emitted with `coefficient: 1` (no effect).
- `spaRenderingCap` does not trigger (same reason). `capScore` is omitted.
- `noViablePathCap` does not trigger because `rendering-strategy.status === "pass"` (one of the four required-non-pass viability inputs is passing). `capScore` is omitted.

Since no cap is triggered, the final `score` equals the base `score` (60).

### B.5 Complete example JSON

```json
{
  "specVersion": "1.0.0",
  "profile": "b2b-saas",
  "scanner": { "name": "Example Conforming Scanner", "version": "1.0.0" },
  "domain": "example.com",
  "finalUrl": "https://example.com/",
  "scannedAt": "2026-04-28T12:00:00.000Z",
  "score": 60,
  "level": 2,
  "levelName": "AI-Aware",
  "nextLevel": {
    "level": 3,
    "levelName": "Agent-Optimized",
    "requirements": ["markdown-negotiation"]
  },
  "categoryReports": [
    {
      "category": "discoverability",
      "description": "Can agents find the signals they need to navigate your site?",
      "score": 64,
      "passed": 1,
      "failed": 1,
      "neutral": 1,
      "checks": [
        {
          "id": "robots-txt-exists",
          "category": "discoverability",
          "status": "pass",
          "scored": true,
          "weight": 7,
          "message": "robots.txt served at /robots.txt",
          "description": "robots.txt is the first file crawlers and agents check for access rules; silence defaults to blanket-allow. Per RFC 9309.",
          "durationMs": 50,
          "specUrls": ["https://www.rfc-editor.org/rfc/rfc9309"],
          "dependsOn": [],
          "details": { "kind": "robots-txt-exists" }
        },
        {
          "id": "sitemap-exists",
          "category": "discoverability",
          "status": "fail",
          "scored": true,
          "weight": 4,
          "message": "No sitemap found at /sitemap.xml or via robots.txt Sitemap directive",
          "description": "An XML sitemap is the route map agents use to find your pages.",
          "durationMs": 120,
          "specUrls": ["https://www.sitemaps.org/protocol.html"],
          "dependsOn": [],
          "details": { "kind": "sitemap-exists" }
        },
        {
          "id": "link-headers",
          "category": "discoverability",
          "status": "neutral",
          "scored": false,
          "weight": 0,
          "message": "Homepage returned no Link header — informational only",
          "description": "Link: response headers expose related resources before HTML is parsed. Per RFC 8288.",
          "durationMs": 0,
          "specUrls": ["https://datatracker.ietf.org/doc/html/rfc8288"],
          "dependsOn": [],
          "details": { "kind": "link-headers" }
        }
      ]
    },
    {
      "category": "accessControl",
      "description": "Are you telling AI systems what they may and may not do with your content?",
      "score": 100,
      "passed": 3,
      "failed": 0,
      "neutral": 1,
      "checks": [
        {
          "id": "ai-bot-rules",
          "category": "accessControl",
          "status": "pass",
          "scored": true,
          "weight": 7,
          "message": "Per-bot rules detected for 3 AI agents",
          "description": "Per-bot robots.txt rules or AIPREF/Content-Signal directives declare who may train on or cite your content.",
          "durationMs": 5,
          "specUrls": ["https://www.rfc-editor.org/rfc/rfc9309"],
          "dependsOn": ["robots-txt-exists"],
          "details": { "kind": "ai-bot-rules" }
        },
        {
          "id": "content-signals",
          "category": "accessControl",
          "status": "pass",
          "scored": false,
          "weight": 0,
          "message": "AIPREF Content-Signal directives present",
          "description": "AIPREF / Content-Signal directives declare what AI systems may do post-fetch. Informational; gate for level 2.",
          "durationMs": 5,
          "specUrls": [
            "https://datatracker.ietf.org/doc/draft-ietf-aipref-vocab/"
          ],
          "dependsOn": ["robots-txt-exists"],
          "details": { "kind": "content-signals" }
        },
        {
          "id": "web-bot-auth",
          "category": "accessControl",
          "status": "neutral",
          "scored": false,
          "weight": 0,
          "message": "Web Bot Auth directory not published — informational only",
          "description": "Operators of AI crawlers MAY publish a signing-key directory at /.well-known/http-message-signatures-directory.",
          "durationMs": 90,
          "specUrls": [
            "https://datatracker.ietf.org/doc/draft-meunier-web-bot-auth-architecture/"
          ],
          "dependsOn": [],
          "details": { "kind": "web-bot-auth" }
        },
        {
          "id": "robots-allow-all",
          "category": "accessControl",
          "status": "pass",
          "scored": false,
          "weight": 0,
          "message": "robots.txt declares wildcard User-agent with explicit Allow: /",
          "description": "A blanket-allow posture declares every crawler is welcome. Informational.",
          "durationMs": 1,
          "specUrls": ["https://www.rfc-editor.org/rfc/rfc9309"],
          "dependsOn": ["robots-txt-exists"],
          "details": { "kind": "robots-allow-all" }
        }
      ]
    },
    {
      "category": "contentReadability",
      "description": "Once an agent is inside, can it actually read what it fetches?",
      "score": 64,
      "passed": 3,
      "failed": 3,
      "neutral": 6,
      "checks": [
        {
          "id": "rendering-strategy",
          "category": "contentReadability",
          "status": "pass",
          "scored": true,
          "weight": 10,
          "message": "Server-side rendering confirmed — agents see content without JavaScript",
          "description": "Classifies the site as SSR, hydrated, or SPA — what agents see without running JavaScript.",
          "durationMs": 800,
          "specUrls": [],
          "dependsOn": [],
          "details": { "kind": "rendering-strategy" }
        },
        {
          "id": "markdown-url-support",
          "category": "contentReadability",
          "status": "neutral",
          "scored": true,
          "weight": 8,
          "message": "Sample inconclusive — could not classify .md twin support",
          "description": "A .md twin alongside each HTML page gives agents an agent-readable fetch path.",
          "durationMs": 400,
          "specUrls": ["https://llmstxt.org/"],
          "dependsOn": [],
          "details": { "kind": "markdown-url-support" }
        },
        {
          "id": "http-status-codes",
          "category": "contentReadability",
          "status": "pass",
          "scored": true,
          "weight": 6,
          "message": "Correct HTTP 404 returned for non-existent path",
          "description": "Soft-404s make agents cache garbage as canonical content. An honest 4xx tells agents the URL is dead.",
          "durationMs": 80,
          "specUrls": [],
          "dependsOn": [],
          "details": { "kind": "http-status-codes" }
        },
        {
          "id": "page-size-html",
          "category": "contentReadability",
          "status": "neutral",
          "scored": true,
          "weight": 6,
          "message": "Sample size insufficient — informational verdict only",
          "description": "Measures how much markdown each page feeds into an agent's context window.",
          "durationMs": 300,
          "specUrls": [],
          "dependsOn": [],
          "details": { "kind": "page-size-html" }
        },
        {
          "id": "markdown-negotiation",
          "category": "contentReadability",
          "status": "fail",
          "scored": true,
          "weight": 5,
          "message": "Server ignored Accept: text/markdown — returned HTML instead",
          "description": "Accept: text/markdown negotiation serves HTML to humans and markdown to agents from one URL.",
          "durationMs": 100,
          "specUrls": [],
          "dependsOn": [],
          "details": { "kind": "markdown-negotiation" }
        },
        {
          "id": "redirect-behavior",
          "category": "contentReadability",
          "status": "fail",
          "scored": true,
          "weight": 4,
          "message": "Sampled URLs use JavaScript redirects — opaque to agents without JS",
          "description": "Same-domain HTTP 3xx redirects work for agents; JavaScript redirects break agents without JS.",
          "durationMs": 250,
          "specUrls": [],
          "dependsOn": [],
          "details": { "kind": "redirect-behavior" }
        },
        {
          "id": "llms-txt-exists",
          "category": "contentReadability",
          "status": "neutral",
          "scored": false,
          "weight": 0,
          "message": "No llms.txt found — informational only",
          "description": "An llms.txt file gives agents a curated entry point into your docs. Per llmstxt.org.",
          "durationMs": 100,
          "specUrls": ["https://llmstxt.org/"],
          "dependsOn": [],
          "details": { "kind": "llms-txt-exists" }
        },
        {
          "id": "llms-txt-valid",
          "category": "contentReadability",
          "status": "neutral",
          "scored": false,
          "weight": 0,
          "message": "Cannot evaluate without llms.txt body",
          "description": "A well-formed llms.txt parses cleanly; a malformed one is skipped silently.",
          "durationMs": 0,
          "specUrls": ["https://llmstxt.org/"],
          "dependsOn": ["llms-txt-exists"],
          "details": { "kind": "llms-txt-valid" }
        },
        {
          "id": "llms-txt-size",
          "category": "contentReadability",
          "status": "neutral",
          "scored": false,
          "weight": 0,
          "message": "Cannot evaluate without llms.txt body",
          "description": "llms.txt must fit in an agent's context window alongside the user's question.",
          "durationMs": 0,
          "specUrls": ["https://llmstxt.org/"],
          "dependsOn": ["llms-txt-exists"],
          "details": { "kind": "llms-txt-size" }
        },
        {
          "id": "llms-txt-has-optional-section",
          "category": "contentReadability",
          "status": "neutral",
          "scored": false,
          "weight": 0,
          "message": "Cannot evaluate without llms.txt body",
          "description": "Reports the shape of your llms.txt — Optional section, H2 count, link count.",
          "durationMs": 0,
          "specUrls": ["https://llmstxt.org/"],
          "dependsOn": ["llms-txt-exists"],
          "details": { "kind": "llms-txt-has-optional-section" }
        },
        {
          "id": "agents-md-detection",
          "category": "contentReadability",
          "status": "fail",
          "scored": false,
          "weight": 0,
          "message": "AGENTS.md not found at /AGENTS.md — informational only",
          "description": "AGENTS.md is a coding-agent convention; presence-only check. Informational.",
          "durationMs": 70,
          "specUrls": [],
          "dependsOn": [],
          "details": { "kind": "agents-md-detection" }
        },
        {
          "id": "cache-header-hygiene",
          "category": "contentReadability",
          "status": "pass",
          "scored": false,
          "weight": 0,
          "message": "Cache-Control header observed on homepage response",
          "description": "Cache-Control, ETag, and Last-Modified let agents re-fetch only what changed. Informational.",
          "durationMs": 0,
          "specUrls": ["https://www.rfc-editor.org/rfc/rfc7234"],
          "dependsOn": [],
          "details": { "kind": "cache-header-hygiene" }
        }
      ]
    },
    {
      "category": "agentEndpoints",
      "description": "Do you expose what agents need to call you programmatically?",
      "score": 0,
      "passed": 0,
      "failed": 4,
      "neutral": 2,
      "checks": [
        {
          "id": "mcp-server-card",
          "category": "agentEndpoints",
          "status": "neutral",
          "scored": true,
          "weight": 10,
          "message": "Detection inconclusive — known well-known paths returned ambiguous shapes",
          "description": "An MCP Server Card advertises your Model Context Protocol endpoint to MCP clients.",
          "durationMs": 250,
          "specUrls": [],
          "dependsOn": [],
          "details": { "kind": "mcp-server-card" }
        },
        {
          "id": "oauth-protected-resource",
          "category": "agentEndpoints",
          "status": "fail",
          "scored": true,
          "weight": 7,
          "message": "No OAuth Protected Resource metadata at /.well-known/oauth-protected-resource",
          "description": "Protected Resource metadata identifies which authorization server protects your API. Per RFC 9728.",
          "durationMs": 90,
          "specUrls": ["https://www.rfc-editor.org/rfc/rfc9728"],
          "dependsOn": [],
          "details": { "kind": "oauth-protected-resource" }
        },
        {
          "id": "oauth-discovery",
          "category": "agentEndpoints",
          "status": "neutral",
          "scored": true,
          "weight": 6,
          "message": "No OAuth surface detected — clean 404 on both well-known paths (hedge)",
          "description": "OAuth discovery metadata at /.well-known/oauth-authorization-server lets agents locate auth endpoints. Per RFC 8414.",
          "durationMs": 180,
          "specUrls": ["https://www.rfc-editor.org/rfc/rfc8414"],
          "dependsOn": [],
          "details": { "kind": "oauth-discovery" }
        },
        {
          "id": "api-catalog",
          "category": "agentEndpoints",
          "status": "fail",
          "scored": false,
          "weight": 0,
          "message": "No /.well-known/api-catalog published — informational only",
          "description": "A /.well-known/api-catalog (RFC 9727) points agents at your OpenAPI specs and developer docs.",
          "durationMs": 90,
          "specUrls": ["https://www.rfc-editor.org/rfc/rfc9727"],
          "dependsOn": [],
          "details": { "kind": "api-catalog" }
        },
        {
          "id": "a2a-agent-card",
          "category": "agentEndpoints",
          "status": "fail",
          "scored": false,
          "weight": 0,
          "message": "No A2A Agent Card at /.well-known/agent-card.json — informational only",
          "description": "An A2A Agent Card describes your service to other agents.",
          "durationMs": 90,
          "specUrls": ["https://a2a-protocol.org/latest/specification/"],
          "dependsOn": [],
          "details": { "kind": "a2a-agent-card" }
        },
        {
          "id": "agent-skills",
          "category": "agentEndpoints",
          "status": "fail",
          "scored": false,
          "weight": 0,
          "message": "No Agent Skills index found at /.well-known/agent-skills/ — informational only",
          "description": "An Agent Skills index exposes your capabilities as discrete skills.",
          "durationMs": 180,
          "specUrls": ["https://agentskills.io/"],
          "dependsOn": [],
          "details": { "kind": "agent-skills" }
        }
      ]
    }
  ],
  "meta": {
    "checksEvaluated": 25,
    "checksSkipped": 0,
    "scanDurationMs": 5000,
    "counters": {
      "fetches": {
        "total": 20,
        "success": 12,
        "wafBlocked": 0,
        "notFound": 8,
        "failed": 0
      }
    }
  },
  "clusters": [
    {
      "name": "htmlPath",
      "kind": "coefficient",
      "triggered": false,
      "coefficient": 1,
      "appliesTo": ["page-size-html"],
      "message": "rendering-strategy passes — page-size-html weight unscaled"
    },
    {
      "name": "spaRenderingCap",
      "kind": "cap",
      "triggered": false,
      "message": "No SPA rendering cap applied"
    },
    {
      "name": "noViablePathCap",
      "kind": "cap",
      "triggered": false,
      "message": "At least one viable agent-content path available"
    }
  ]
}
```

---

## Annex C — Implementation notes

_(Informative, non-normative.)_

This annex collects guidance for scanner implementers. Nothing in this annex is normative; conformance is judged against [§11](#11-conformance).

### C.1 HTTP probe conventions

- Implementations SHOULD use a unique `User-Agent` string identifying the scanner. This allows site operators to identify scanner traffic in their logs and apply appropriate policies.
- Implementations SHOULD respect `robots.txt` for non-well-known paths during catalog probes. Probes of `/.well-known/*` paths and `robots.txt` itself need not be gated by `robots.txt` (those paths are by convention publicly probable; reading `robots.txt` is the prerequisite to respecting it).
- Per-request timeouts and retry policy are implementation-defined. Implementations SHOULD apply reasonable timeouts so that a slow check does not block the entire scan indefinitely.
- Implementations SHOULD cap pre-parse response body sizes at a reasonable threshold (a 2 MB cap is a reasonable default for `sitemap-exists`'s XML parsing; other checks may apply different caps appropriate to their content type).
- Implementations SHOULD apply per-host rate-limit politeness (a small minimum gap between successive requests to the same origin) to avoid triggering site-side rate limiting.

### C.2 Headless rendering

- The `rendering-strategy` check requires comparing plain HTML to a JavaScript-rendered version of the homepage. Implementations MAY use any headless browser tooling for this purpose.
- The neutral-when-unavailable path on `rendering-strategy` (returning `neutral` when headless rendering is unavailable AND the plain body is not detected as an SPA shell) exists specifically to accommodate environments without rendering capability. Implementations without rendering MUST NOT emit `fail` in that case; they MUST emit `neutral`.
- When `rendering-strategy` returns `neutral`, the `htmlPath` and `spaRenderingCap` clusters MUST NOT trigger (their trigger conditions reference `rendering-strategy.status === "fail"`).

### C.3 WAF and rate-limit handling

- For `markdown-negotiation`, HTTP `403` and `429` responses are NORMATIVELY mapped to `neutral` (treated as WAF or rate-limit interference rather than a genuine fail signal). See [§5.3](#53-contentreadability) for the precise predicate.
- For other checks where similar interference may occur, implementations SHOULD prefer `neutral` over `fail` when the observation is plausibly tainted by access-control or rate-limit responses outside the site's normative agent-readiness posture.
- Implementations MAY surface the underlying cause (e.g., specific HTTP status, network timeout) inside `details` for debugging purposes, but the wire `status` field MUST reflect only one of the three enum values.

### C.4 Sampling

- Several checks sample multiple URLs from the sitemap or homepage rather than probing a single endpoint. v1.0 catalog entries that sample include `markdown-url-support` (up to 5 URLs), `redirect-behavior` (up to 5 URLs), and `page-size-html` (up to 10 URLs).
- The minimum-sample-size thresholds for pass/fail evaluation declared in [§5](#5-checks-catalog) are NORMATIVE (e.g., `markdown-url-support` requires at least 3 measurable probes before emitting pass/fail; insufficient samples emit `neutral`).
- The selection strategy WITHIN those sample-size bounds is implementation-defined (e.g., random sampling, deterministic by sitemap order, prioritizing recently-modified URLs). Implementations SHOULD document their sampling strategy if reproducibility across runs is a requirement.

### C.5 Per-check `details` shape

- The `details` payload for each check is implementation-defined within the per-check semantic. The Specification does NOT prescribe an exhaustive `details` schema for any check.
- Consumers MUST parse `details` defensively, treating any field other than `kind` as potentially absent or implementation-specific.
- Setting `details.kind` equal to the enclosing check's `id` is RECOMMENDED for consumer-side discrimination of the typed payload.
- `details.body` is FORBIDDEN regardless of mode. Implementations needing to surface raw response bodies for diagnostics MUST do so out-of-band (e.g., in side-channel logs or debug-mode artifacts) — never on the v1.0 normal-mode wire.

### C.6 Per-host courtesy

- Implementations SHOULD avoid excessive concurrency against a single origin. A single scan typically issues fewer than 50 fetches against the target origin; with appropriate per-host rate limiting this is well within reasonable courtesy.
- Implementations SHOULD honor `Retry-After` response headers on `429` and `503` responses where retry is attempted at all.

### C.7 Mapping runtime errors to status

When an unexpected error occurs during a check (uncaught exception, malformed response, parse failure):

- If the check executed enough to evaluate against pass criteria but the result is unusable, implementations SHOULD emit `fail` with a `message` describing the runtime issue.
- If the observation could not be made at all (network failure, dependency unavailable, infrastructure missing), implementations SHOULD emit `neutral` with a `message` describing why no observation was made.
- Either way, the status field MUST be one of the three enum values; v1.0 has no `error` status.
