---
name: Bug report
about: Report something the skill got wrong (false positive, hallucinated finding, broken workflow)
title: "[bug] "
labels: bug
---

## What happened

<!-- One paragraph. What did the audit report claim, and what did you observe in your project? -->

## Expected behavior

<!-- What should the skill have done instead? -->

## Reproduction

- **Skill version / commit:** <!-- `git -C ~/.claude/skills/rails-audit log -1 --oneline` -->
- **Mode:** <!-- quick / standard / deep -->
- **Rails version:** <!-- `bundle exec rails -v` -->
- **Ruby version:** <!-- `ruby -v` -->
- **Deploy target:** <!-- cloud_run / heroku / kamal / etc. -->
- **Job adapter:** <!-- sidekiq / cloudtasker / etc. -->

## The finding the skill produced

<!-- Paste the exact finding entry from the report — including file path, line number, and severity. -->

## What's actually true

<!-- Paste the relevant code/config that contradicts the finding. -->

## Anything else

<!-- Tooling that was missing? `.claude/rails-audit.yml` contents? -->
