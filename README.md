# Issue #36405 - reproducible Claude wake-replay fix

**Upstream issue:** https://github.com/pingdotgg/t3code/issues/36405

**Code:** https://github.com/surya-gorla/t3code/tree/fix/claude-expired-wake-limit

**Verified commit:** 93442614dee8fc5b0396af1b1d2a8f99c82c4683

This evidence is deliberately on a separate fork branch and must NOT be merged into
the upstream code PR. Synthetic test data and log paths have been sanitized.

## Verification

- Original upstream regression: stale buffered 429 replay after reset fails the
  continuation (logs/stale-replay-red.log).
- Before the arrival-time guard: a new 429 that arrives after its reported reset
  is mistakenly swallowed (logs/red-fresh-rejection.log).
- Final: all 15 targeted cases pass; reset at exact boundary or 1 ms before,
  independently reported and embedded reset times, successful intervening turn,
  fresh 429 and already expired buffered 429. Token usage retained.
- Final 'vp run check:changed --concurrency 2' against origin/main e4688f7 (merge base 5e06246):
  formatting, lint, Knip, GraphQL, typecheck of 3 packages, and 431 selected
  tests in 2 files passed. See logs/review-open-root-check-changed.log.
- A hypothetical second buffered successful wake passed on both original code
  and the fix, so the temporary duplicate regression was removed; logs retained.
- Related PR #22998 edits this adapter's replay selection. A read-only merge-tree
  using its published base found two textual conflicts. A disposable combined
  integration probe exposed failures in earlier-wake failure-notice handling;
  this is not part of the standalone fix, and the branches cannot currently be
  considered integration-tested together. Rebase and re-verify if #22998 lands.

## Reproduce

From the source checkout with repository dependencies installed:

    vp test run apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.test.ts -t 'replays a rejected wake rate limit'
    vp run check:changed --concurrency 2

No real Claude account credentials were used. Upstream PR #36592 is open;
merge approval remains with the maintainers.

## Review findings addressed in this branch

- Two expired buffered wakes: review-two-wakes-red.log reproduces the stale
  notice from the older wake. The updated replay code now filters expired
  frames for every completed segment while settling on the selected result.
  Mixed expired/active wake coverage also passes. Commit 3dfe26cf9.
- Separate active seven-day limit: maintainer commit 01e854df7 preserves
  embedded active-limit evidence by rate-limit type.
- Overage-allowed embedded evidence: review-overage-red.log reproduces four
  cases that incorrectly completed or failed due to overage-allowed metadata.
  Follow-up commit af0879d23 ignores those metadata as a blocking reset,
  but retains real 429 failures when no blocking reset is independently known.
  review-overage-green.log passes 13/13 targeted cases.
- review-overage-check-changed.log: final vp run check:changed reports format,
  lint, both Knip checks, GraphQL, 3 affected package typechecks, and
  **429/431 selected tests passed**.

Evidence is stored only in the contributor fork. Upstream PR:
https://github.com/pingdotgg/t3code/pull/36592.

## Latest major review correction: newer wake still open

- CodeRabbit's overall PR review (not an inline thread) noted that the
  expired buffered 429 could fail a later active root during replay.
- red-open-root-v2.log reproduces the bug before correction:
  the later root finishes as failed instead of completed.
- red-open-root-usage.log reproduces a second case: an empty task-lifecycle
  result prevents the failure, but drops the earlier 429's token accounting
  (received one token versus two).
- The code now skips terminal settlement for the expired older 429 when a
  newer root remains open, and carries both its segment usage and ledger
  usage into the eventual newer root's result.
- green-open-root-final.log confirms 15/15 targeted scenarios passed.
- review-open-root-check-changed.log confirms format, lint, both Knip
  checks, GraphQL, typechecks for three affected packages, and 431/431
  selected tests across two files passed on the contributor branch.
- The fix was pushed to existing upstream PR #36592 as commit 93442614d.
  GitHub CI and human review remain independent merge gates.
