---
name: Dimension or check request
about: Suggest a new dimension, a check within an existing dimension, or a stack profile (Kamal, Fly, ECS, etc.)
title: "[dimension] "
labels: enhancement
---

## What should the skill catch that it doesn't

<!-- Concrete example: "the skill missed that Sidekiq Web UI was mounted without auth" or "no check for Postgres pgbouncer transaction-mode incompatibility" -->

## Failure mode in production

<!-- What real-world incident or near-miss would this catch? Skills that catch hypothetical problems get noisy fast — anchor it in something that has bitten or could realistically bite. -->

## Where it fits

- **Dimension:** <!-- spec-and-coverage / deploy-ci / security-and-authz / money-and-payments / code-health / performance-reliability / background-jobs / observability / data-integrity / data-governance / dx-and-cost / new -->
- **Tier:** <!-- Blocker / High / Medium / Low — and the rationale -->
- **Detection:** <!-- grep pattern, tool that finds it, manual read required, etc. -->

## Stack-specific?

<!-- If this only applies to Heroku / Cloud Run / Kamal / a specific job adapter, call that out so the check can be scoped. -->
