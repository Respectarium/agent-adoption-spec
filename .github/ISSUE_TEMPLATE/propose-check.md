---
name: Propose a check
about: Suggest a new check or a modification to an existing check
title: "[Propose] "
labels: proposal, check
---

## What check are you proposing?

A short description of the check.

## Proposed identity

- **Proposed `id`:** (lowercase kebab-case, e.g., `markdown-url-support`)
- **Proposed `category`:** one of `discoverability`, `accessControl`, `contentReadability`, `agentEndpoints`
- **Proposed `weight`:** an integer from `{0, 4, 5, 6, 7, 8, 10}` (`0` = informational; `4`–`10` = scored, contributes to score per SPEC.md §7)

## What does it measure?

What aspect of agent-readiness does this check measure that existing checks don't already cover?

## How would a scanner test it?

Describe the HTTP-level protocol (request method, path, headers, expected response shape). Pseudocode is fine. Indicate any dependencies on other checks.

## Pass / fail / neutral criteria

- **Pass**: what conditions in the response → `pass` status
- **Fail**: what conditions → `fail`
- **Neutral**: under what conditions does this check return `neutral` (not applicable, observation could not be made, dependency-skip, etc.)?

## Empirical basis

Is there research showing this signal correlates with agent or LLM behavior? If yes, link or summarize. If not, the check may still be valid for inclusion as informational (`weight: 0`); scored weight (`4`–`10`) is reserved for checks with empirical support.

## References

- Relevant RFCs, drafts, vendor docs, prior art
