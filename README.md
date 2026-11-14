# Issue #36405 - reproducible Claude wake-replay fix

**Upstream issue:** https://github.com/pingdotgg/t3code/issues/36405

**Code:** https://github.com/surya-gorla/t3code/tree/fix/claude-expired-wake-limit

**Verified commit:** e9e89eada1c66f851c7214fc8be2eb2612b57d3d

This evidence is deliberately on a separate fork branch and must NOT be merged into
the upstream code PR. Synthetic test data and log paths have been sanitized.

## Verification

- Original upstream regression: stale buffered 429 replay after reset fails the
  continuation (logs/stale-replay-red.log).
- Before the arrival-time guard: a new 429 that arrives after its reported reset
  is mistakenly swallowed (logs/red-fresh-rejection.log).
- Final: all 8 targeted cases pass; reset at exact boundary or 1 ms before,
  independently reported and embedded reset times, successful intervening turn,
  fresh 429 and already expired buffered 429. Token usage retained.
- Final 'vp run check:changed --concurrency 2' on upstream main 62ba72fa:
  formatting, lint, Knip, GraphQL, typecheck of 3 packages, and 421 selected
  tests in 2 files passed. See logs/check-changed-rebased-guard.log.
- A hypothetical second buffered successful wake passed on both original code
  and the fix, so the temporary duplicate regression was removed; logs retained.
- Local direct merge check with open PR #22998 could not establish a merge base
  because this checkout is shallow. Their PR touches the same server adapter and
  test file; no conflict-free integration claim.

## Reproduce

From the source checkout with repository dependencies installed:

    vp test run apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.test.ts -t 'replays a rejected wake rate limit'
    vp run check:changed --concurrency 2

No real Claude account credentials were used. No upstream pull request was
created for issue #36405; this is a fork branch pending further integration
review and explicit developer authorization.
