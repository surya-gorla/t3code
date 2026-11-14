# T3 Code #36405 - Expired buffered Claude wake limit

- Repository: pingdotgg/t3code
- User requested a substantial bug contribution under the repository's rules. All work is isolated to this workspace.
- The current upstream contribution policy allows only bug fixes from external contributors, one issue per PR, a three-open-PR maximum, no PR farming or irrelevant scope, and verified failure before fix. Existing user PRs: #36476 and #36494, both open and in maintainer takeover. No new upstream PR is authorized by this log.
- Bug issue: https://github.com/pingdotgg/t3code/issues/36405. An old buffered Claude 429 wake result is replayed after its stated reset, causing a later queued continuation to fail again with a stale usage limit.
- Reproduced before change against main fcf50ffe: `stale-replay-red.log` records expected 'completed', received 'failed'.
- Local fix confined to apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.ts and matching existing test. It identifies explicitly expired rate-limit data for selected buffered wake result, avoids reapplying its old rejected event/assistant frames, and settles the continuation while preserving result usage. Live and still-valid failures continue down existing paths.
- Expanded deterministic shared adapter tests: 5/5 targeted tests pass (`stale-replay-intervening.log`), including original standalone/embedded still-valid rejection, expired standalone/embedded rejection, and queued continuation after a newer successful user turn.
- Latest local `vp run check:changed --concurrency 2` passed: formatter, linter, Knip halves, GraphQL check, typecheck 3 packages, 418 tests over 2 files (`check-changed-intervening.log`). `git diff --check` clean.
- Related PR #22998 is open and changes the same Claude adapter and test files to fix a different multi-wake replay issue; it is labelled needs-rebase. Review integration/merge overlap before opening an upstream PR.
- Open questions before submission: inspect interactions with #22998, preserve billed usage and lack of stale usage events, and rebase/test on latest upstream main. Live signed-in Claude provider integration was not run.
- The user's request was to select a more substantial issue, not to grow diffs for size metrics. Keep this fix focused; no unsolicited maintainer pings. A fork backup branch is not an upstream PR.

## Independent hardening (14 Nov 2026)

- Freshened origin/main to 62ba72fa15441050469740a51a19b45a8b1085c0, rebased local fix cleanly.
- Added time-boundary cases: exactly at the published reset (success) and one millisecond before (still rejects). Both passed.
- Added a counterexample where the provider's 429 result ARRIVES AFTER its advertised reset: initial implementation incorrectly marked this real new error completed; reproduced red in red-fresh-rejection.log. Corrected by using the existing bufferedResultArrivals WeakMap: only suppress a buffered rejection if its result arrived before the published reset and replay occurs after it.
- Tested synthetic conflicting data: standalone rate-limit event is authoritative over embedded assistant info; the old still-active rejection remains a failure.
- Existing wake-turn test checks successful intervening normal user turn before delayed continuation, no stale system_notice, and recorded input/output token usage for stale wake.
- New targeted run: 8/8 cases pass (green-arrival-guard.log, qa-usage-preservation.log). Full changed-file run against rebased main: 421 tests in 2 selected files pass, typecheck of 3 affected packages, format, lint, Knip 1/2 and 2/2, GraphQL check (check-changed-rebased-guard.log). git diff --check clean.
- Also wrote and exercised an additional two-wake success scenario: it passed on our code AND on the unmodified baseline, so we removed that redundant 103-line test from the commit in keeping with upstream's lowest-layer and minimal-test rules. Logs retained locally (multi-wake-baseline.log).
- Inspected overlap with upstream PR #22998: it touches the same two adapter files and modifies replay bookkeeping. That PR is still open, labelled needs-rebase. We tried an isolated read-only merge check, but shallow history prevented merge-base resolution. DO NOT claim conflict-free integration with #22998. No PR opened for issue #36405.
- Amended the candidate to one focused code commit e9e89eada1c66f851c7214fc8be2eb2612b57d3d, 2 files changed, 209 insertions, 9 deletions. Pushed with an exact --force-with-lease to https://github.com/surya-gorla/t3code/tree/fix/claude-expired-wake-limit; remote head verified equal to e9e89ead and local worktree clean. Existing upstream PRs #36476 and #36494 not touched.
- No live authenticated Claude session exercised, as this would consume real account quota and private data. Synthetic Effect-clock tests validate deterministic protocol event behavior.
- Remaining next actions: consider additional official CI on a PR only after explicit developer request per AGENTS.md; re-review overlap if #22998 changes or merges; do not create unrelated feature changes or additional unsolicited PRs.
