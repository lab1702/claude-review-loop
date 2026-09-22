---
name: review-loop
description: Run independent whole-repository reviews, fix verified issues, run checks, and make authorized local commits.
argument-hint: I authorize ordinary commits to the current branch.
disable-model-invocation: true
---

# Review Loop

Review the whole repository and fix verified issues until two consecutive passes are clean on the same unchanged commit. Attempt at most 10 passes.

Each pass runs these steps in order:

1. [Launch reviewer](#launch-reviewer): a fresh subagent reviews the expected local HEAD.
2. [Assess review](#assess-review): accept or reject the completed review, then retire the reviewer.
3. [Validate and fix findings](#validate-and-fix-findings).
4. [Check and stage content](#check-and-stage-content): if no content changed, the pass ends here.
5. [Commit fixes](#commit-fixes).
6. [Verify commit](#verify-commit).

[Completion and limits](#completion-and-limits) then decides whether to finish, stop, or start the next pass.

## Terms

- **Stop**: end the entire run as **blocked** and follow [Exit and restart](#exit-and-restart).
- **Starting branch**: the branch checked out during preparation.
- **Expected local HEAD**: the commit HEAD must match. It starts as the starting branch's commit and advances after each verified commit.
- **Content changes**: edits, additions, or deletions of tracked or non-ignored untracked files, excluding run artifacts removed as described in [Check execution rules](#check-execution-rules) or [Validate and fix findings](#validate-and-fix-findings).
- **Run artifact**: a non-ignored untracked file or directory that did not exist before a check run, validation run, or commit, was created during it by a prerequisite setup step, check command, validation command, or hook, and is not part of a verified fix (for example, a coverage report, a linter cache, or package build metadata). A check run here includes its prerequisite setup and any confirmation rerun of it.
- **Allowed check-induced change**: a change a prerequisite setup step, check, or hook makes within a verified fix, or a formatting-only change it makes elsewhere in a file that a verified fix touched (for example, a formatter reformatting fixed code or the rest of its file). Allowed check-induced changes are part of the fix, not unrelated work.
- **Clean working tree**: no staged changes, unstaged changes, or non-ignored untracked files.
- **Full suite**: all currently required or selected relevant checks, initially identified during preparation.
- **Check run**: a full suite or a targeted check.
- **No-checks exception**: recorded when no checks are required and no relevant checks exist. It waives check execution only.
- **Confirmation rerun**: a targeted check that reruns only the failed check commands, unchanged, to test whether a failure is flaky.
- **Failure-repair attempt**: diagnosing a check failure using only existing output (including any confirmation rerun) and static inspection, then repairing a verified repository issue.
- **Stabilization rerun**: the first full suite after a check run changes content, counting only check runs since the start of the pass or your latest edit to content (a fix or a failure-repair attempt).
- **Clean pass** and **non-clean pass**: defined in [For each review pass](#for-each-review-pass).

## Launch requirements

Require explicit authorization for local commits on the current branch, for example: "I authorize ordinary commits to the current branch." Authorization covers the entire run; invocation alone is insufficient. If it is missing or unclear, request it and wait.

Launch each reviewer with the Agent tool as a new, non-fork subagent of type `review-loop:reviewer`, this plugin's read-only reviewer, which starts with only the prompt you give it. If that type is unavailable, use `general-purpose` and note the fallback in the final report. Do not use a `fork` subagent, continue an earlier reviewer with `SendMessage`, or shell out to `claude -p` or another CLI to obtain isolation. An instruction to "ignore previous context" does not establish isolation; sharing the repository filesystem is allowed. If the Agent tool is unavailable, stop.

## Run boundaries

- Network access is allowed as needed for review, repairs, and checks, including documentation lookups, GitHub queries, dependency downloads, and fetching missing Git objects. Fetches, including automatic fetches in partial clones, must preserve the checked-out branch, HEAD, index, and working-tree content.
- Leave all Git pushes to the user. Do not push to GitHub or any other remote, directly or through reviewers, checks, or hooks.
- Do not amend commits, rewrite history, create or switch branches, discard work from outside this run, include unrelated work, or weaken tests/checks. You may revise or revert your own uncommitted fix attempts during the run.
- Temporary files and directories are allowed for review, analysis, repairs, and checks. Keep scratch files and review artifacts (transcripts, finding inventories, and run logs) outside the working tree or in an ignored location; artifacts may also stay in the conversation. Never commit them. Clean up disposable files created by the run when they are no longer needed.

Before each reviewer launch, edit, or commit, and at completion, verify that the starting branch is checked out and HEAD matches the expected local HEAD. Stop on a mismatch, or if any working-tree change is unexplained or comes from outside this run.

Require no merge, rebase, cherry-pick, revert, or other sequencer operation in progress during preparation and before every commit. For at least `MERGE_HEAD`, `CHERRY_PICK_HEAD`, `REVERT_HEAD`, `rebase-merge`, `rebase-apply`, and `sequencer`, resolve the path with `git rev-parse --git-path <name>` and confirm it does not exist; a clean working tree alone is insufficient. Stop without completing or aborting an existing operation.

## Preparation

- Follow applicable user and repository instructions, including CLAUDE.md files.
- Require a clean working tree with a valid HEAD on a checked-out branch; otherwise stop.
- Record the starting branch and set the expected local HEAD to its commit. Any branch, including `main`, is supported; no remote or upstream is required.
- Identify required check commands (tests, lint, type checking, and builds). If none are specified, select relevant available checks and state their scope.
- If commit hooks are configured (for example, through `core.hooksPath`, the hooks directory from `git rev-parse --git-path hooks`, or a framework such as pre-commit, husky, or lefthook), include them in the full suite as hook checks, limited to files changed in the pass. Skip hook checks when no files changed.
  - If the framework has its own command that accepts files (for example, `pre-commit run --files <changed files>`, or `lefthook run pre-commit --no-auto-install --no-stage-fixed --file <file>` with `--file` repeated for each changed file), run it with the other checks. This surfaces hook failures and hook reformatting before staging.
  - Otherwise, run the configured `pre-commit` hook as the last command of the full suite, with exactly the verified fixes staged: `git hook run --ignore-missing pre-commit`. Staging for this check is not a content change; restage after any repair or check-induced change before rerunning it.
  - If a `commit-msg` or `prepare-commit-msg` hook is configured, also check the planned commit message, kept in a message file outside the working tree. Run the message hooks only on a fresh copy of that file, never on the file you will pass to `git commit -F`, because hooks may edit the message and the commit runs them again. If a `prepare-commit-msg` hook is configured, run it on the copy with `git hook run --ignore-missing prepare-commit-msg -- <copy> message`, so any later check sees the message as the commit will produce it. If a `commit-msg` hook is configured, then run `git hook run --ignore-missing commit-msg -- <copy>`. A failed message check is handled as described in [Check execution rules](#check-execution-rules).
  - If Git is older than 2.36 and lacks `git hook run`, execute the hook file from the hooks directory instead.
- Inspect the hooks that a commit triggers, including `post-commit` and hooks managed by a framework, and stop if any of them would push to a remote.
- If no checks are required and no relevant checks exist, record a no-checks exception.
- Unless the no-checks exception applies, run the baseline suite: the full suite against the starting commit, before any review, following [Check execution rules](#check-execution-rules). Hook checks are skipped because no files changed. The baseline allows no repairs, and any content change other than removed run artifacts stops the run.
  - If a check fails without content changes, run one confirmation rerun. Record each check that then passes as flaky, with its failing output, for the final report.
  - Stop if any check still fails, reporting its output as a failure that already existed at the starting commit, so the user can fix it or its prerequisites before a new run.
  - Record a baseline that passes without content changes, directly or with its failed checks passing on the confirmation rerun, as a passing full suite for the starting commit's tree, noting any flaky checks.
- Initialize the attempted-pass and consecutive-clean counters to zero.

## Reviewer prompt

Give each reviewer only the prompt below with its placeholders filled in. Summarize user requirements without relying on prior conversation; use "None specified" if there are none. Do not attach prior findings, fix explanations, or this skill.

```text
Repository: <repository location>
Commit to review: <exact commit SHA>
Applicable project requirements: <requirements or their locations>
Applicable user requirements and constraints: <self-contained summary or None specified>

Review the whole codebase at this exact commit, including source, tests,
configuration, and scripts, not only a diff. The working tree is clean at
this commit, so you may read tracked files directly; ignore untracked and
ignored files. Follow applicable project instructions.

Inventory generated files, vendored dependencies, binaries, and submodules.
Review their integration and relevant correctness or security risks; inspect
submodules at their recorded commits, fetching missing objects if needed.
You may omit detailed inspection of generated or vendored content when
reviewing its maintained inputs or integration is sufficient.

Use static inspection of committed content. Network lookups, GitHub queries,
and Git fetches (including automatic fetches in partial clones) are allowed
as needed. Preserve the checked-out branch, HEAD, index, and working-tree
content when fetching. If required objects remain unavailable, report the
affected paths and coverage gap. Never push to GitHub or any other remote;
the user handles all pushes.
You may create temporary files, directories, and inspection helpers outside
the working tree or in an ignored location. Never stage or commit them, and
clean up disposable files you created when finished. Do not execute tests,
builds, or repository scripts; the coordinator runs checks. Do not consult
earlier review artifacts, edit repository content or the index, switch or
create branches, or create or alter commits.

Report concrete, actionable findings with file/line references, triggering
scenarios, and impact. Do not request cosmetic changes, speculative
refactoring, or hardening without a demonstrated failure. If you find no
actionable issues, say so explicitly.

State what you reviewed. For each excluded or unavailable category or path,
give the reason and any residual coverage gap with its potential impact.
A material coverage gap prevents a supported conclusion about the correctness
or security of an in-scope component or behavior.
```

## Check execution rules

Use network access and temporary environments to obtain routine prerequisites when needed. Prefer setup that leaves tracked files unchanged: lockfile-respecting, frozen installs (for example, `npm ci` or `uv sync --frozen`) and environments outside the working tree or in ignored locations. Stop if a required runtime, service, or other prerequisite remains unavailable.

Reassess the check commands and any no-checks exception at the start of each pass and after changes to tests, check configuration, dependencies, or project instructions, including changes made by checks or hooks. Include newly available or required checks, and revoke the exception when checks now exist or are required. If the suite changes, invalidate earlier results and require the updated full suite before staging or completing the pass. Changes that hooks make during a commit are handled in [Verify commit](#verify-commit) instead. This does not reset repair or stabilization limits.

Before staging or completing a pass with an accepted review, require a passing full suite that leaves content unchanged, unless the no-checks exception applies. Results apply only to the exact content they ran against. If content matches the expected local HEAD's tree, you may reuse a passing full suite recorded earlier in the run for that tree, including results applied in [Verify commit](#verify-commit), instead of rerunning it, provided the full suite has not changed since. Record each reuse and the pass, or the baseline suite, whose results were reused.

Compare repository status and content before and after every prerequisite setup step and check command, regardless of exit status, and note any run artifacts it created. Keep run artifacts until the check run, including any confirmation rerun of it, finishes, so later commands can use earlier outputs; then delete them, or delete them before stopping, and record their paths for the final report. Removed run artifacts are not content changes. Other setup- or check-induced changes must be allowed check-induced changes; stop on any other change, including any change to tracked files that a verified fix did not touch.

The following recovery rules apply only before committing. Post-commit checks follow [Verify commit](#verify-commit).

- A failure caused only by allowed check-induced changes, such as a formatter hook that exits with an error after reformatting files, needs no repair and is not a failure-repair attempt. Confirm this from the check output before treating the failure that way.
- Allow at most two failure-repair attempts per pass. Stop if any other failure has no verified repair or would require a third attempt. Count every repair prompted by a check failure, even if the reviewer also reported the issue; fixes made only for reviewer findings do not count.
- Once the stabilization rerun starts, any further check-induced content change stops the run, until your next edit to content (a fix or a failure-repair attempt) resets stabilization status to not started. The failure-repair limit and the stop for attempts that repeat without progress still bound the pass.
- Allow at most one confirmation rerun per pass. It does not count as a failure-repair attempt. Record each failed check that passes on the rerun without content changes as flaky, with its failing output, for the final report, even if other checks fail again, but only when the confirmed check run made no content changes; otherwise record it as passing after check-induced changes; a flaky failure alone does not verify a finding or make the pass non-clean. If every failed command passes on the rerun without content changes, the check run it confirmed counts as a success, keeping any content changes that run made; a full suite that passes this way without content changes is a passing full suite.
- A failed `commit-msg` or `prepare-commit-msg` check does not follow the table below. Revise the planned message file, then rerun only the message checks; other results still apply because content is unchanged. This is neither a confirmation rerun nor a failure-repair attempt. Stop if no accurate summary of the verified fixes satisfies the hook.

Apply the table after all commands in the check run finish, subject to the limits above, unless a stop condition requires immediate exit:

| Check-run result | Required next action |
| --- | --- |
| Success without content changes | Full suite: proceed. Confirmation rerun: treat the check run it confirmed as a success and apply the matching row to it. Message-check rerun: earlier results still apply. Other targeted check: run the full suite before staging or completing the pass. |
| Failure without content changes | If the pass's confirmation rerun is unused, run it next. Otherwise, perform one failure-repair attempt, then immediately run the full suite. |
| Success with content changes | Run the stabilization rerun next. |
| Failure with content changes | If the failure was caused only by allowed check-induced changes, run the stabilization rerun next. Otherwise, if the pass's confirmation rerun is unused and every other failure came from a check that made no content changes, run it next, keeping the check-induced changes. Otherwise, perform one failure-repair attempt, then immediately run the full suite. |

## For each review pass

Mark the pass permanently **non-clean** and reset the consecutive-clean count to zero as soon as any of these events occurs:

- Reviewer launch or acceptance fails.
- A finding is verified, including during checks.
- Content changes, even if later reverted. A regression test written only to validate a finding that is then rejected, and removed afterward, does not count.

Otherwise, the pass is **clean** once [Check and stage content](#check-and-stage-content) succeeds. Rejected findings alone do not disqualify it.

### Launch reviewer

At the start of each pass, reset failure-repair attempts to zero, the confirmation rerun to unused, and stabilization status to not started. Increment the attempted-pass count, then launch a fresh reviewer of the expected local HEAD with the [Reviewer prompt](#reviewer-prompt), as required in [Launch requirements](#launch-requirements). Run it in the foreground (`run_in_background: false`) so its result returns before you continue.

### Assess review

Wait for the reviewer to finish before editing. Confirm that the starting branch is still checked out, HEAD matches the expected local HEAD, and the working tree and index are unchanged; otherwise stop. Accept only a completed review with no unresolved execution errors or material coverage gaps, as defined in [Reviewer prompt](#reviewer-prompt).

Then [retire the reviewer](#retire-reviewer), whether or not the review was accepted.

If launch or acceptance fails and the problem is likely to clear up in another pass (for example, a timeout or a reviewer that ended early), end the pass as non-clean and continue at [Completion and limits](#completion-and-limits). Otherwise stop.

#### Retire reviewer

A foreground reviewer has finished once its result returns. If a reviewer is still running anyway (for example, because it was moved to the background), stop it with `TaskStop`. Never reuse or resume it, including with `SendMessage`.

### Validate and fix findings

Validate each finding against the reviewed commit and record reasons for rejections. Verify a finding only when the reviewed commit demonstrably has the defect: a concrete input, state, or sequence of events that causes incorrect behavior, a failure, or a security exposure, or content that violates an applicable project or user requirement, including documentation that contradicts actual behavior. Confirm it by inspecting the code and, where practical, with a reproduction kept outside the working tree or a regression test. Running a reproduction or regression test to confirm a finding is a validation run, not a check run: its expected failure is not a check failure and uses neither the confirmation rerun nor a failure-repair attempt. Delete run artifacts a validation run creates as soon as it finishes, before any check run, and record their paths for the final report. If the finding is rejected, remove any regression test written only to validate it. Reject style or naming preferences, speculative refactoring or hardening without a demonstrated failure, and claims that rely on unsupported assumptions. Apply the same standard in every pass.

Resolve all verified findings, adding regression tests where appropriate.

Stop if:

- Required information or a consequential choice cannot be inferred, or required authorization is missing.
- A verified finding cannot be resolved within scope.
- No evidence-backed next step remains, or attempts repeat without progress.

### Check and stage content

Satisfy [Check execution rules](#check-execution-rules) before proceeding.

If no content changes remain relative to the reviewed commit, unstage anything staged for hook checks (for example, `git restore --staged :/`) so the working tree is clean, and skip staging and committing; the pass ends here. Continue at [Completion and limits](#completion-and-limits).

Otherwise, stage only verified fixes and leave no unstaged tracked changes or non-ignored untracked files. Unless the no-checks exception applies, confirm that content has not changed since the passing full suite, so the staged tree is exactly what passed. Record the staged tree ID (`git write-tree`) with the check results or exception. Keep content unchanged until committing.

### Commit fixes

Verify that the staged tree still matches the recorded tree ID, then commit to the starting branch. Make one commit per pass whose message summarizes the verified fixes, following the repository's commit conventions. If a message check ran, commit with the message file (`git commit -F <message file>`), not the copy, so each message hook edits the committed message only once.

If the commit command fails, including hook rejection, delete run artifacts that the commit's hooks created and record their paths, inspect HEAD, the index, and working tree for the final report, then stop without repair, retry, or bypassing hooks.

### Verify commit

Delete run artifacts that the commit's hooks created, and record their paths for the final report. Then require the starting branch to remain checked out, a clean working tree, and a new commit whose sole parent is the expected local HEAD and whose changes are all intended. Compare its tree ID (`git rev-parse 'HEAD^{tree}'`) with the recorded staged tree ID:

- If they match, the recorded check results or no-checks exception apply.
- If they differ, verify that hooks caused the differences and that they are allowed check-induced changes, then run a full suite against the new commit unless the no-checks exception applies. Omit hook checks from this suite: the commit's hooks already ran on its content, and nothing remains staged for them to check.

Stop without repair or retry if verification fails, including any post-commit check failure or content change. On success, advance the expected local HEAD to the new commit.

## Completion and limits

At the end of each pass, increment the consecutive-clean count if clean, then evaluate in order:

- If two consecutive passes are clean, verify that both reviewed the same unchanged commit and the working tree is clean. Finish as **completed** if verified; otherwise stop.
- If 10 passes have been attempted, stop with the reason **review-pass limit reached**. Otherwise start the next pass.

## Exit and restart

On every exit, retire any remaining reviewer, preserve local commits and uncommitted changes, and produce the [Final report](#final-report).

A new invocation restarts at [Launch requirements](#launch-requirements) with fresh counters. The user must first resolve any blockers and ensure a clean working tree. If the only stop reason was the pass limit, the user can re-run right away.

## Final report

- Outcome: completed or blocked. If blocked, explain the stop reason and any prerequisites for a new run.
- Starting branch and final commit. Mark unavailable or unverified Git values explicitly and explain why.
- Fixes, rejected findings with the reasons for rejecting them, the baseline suite result, checks and their results (or the no-checks exception), reused check results, checks found flaky with their failing output, checks that passed after check-induced changes, review coverage, and remaining limitations.
- Run artifacts deleted during the run, with a suggestion to add ignore rules for them, and any reviewer agent fallback.
- Attempted review passes and consecutive clean passes.
- Local commits created during the run, any uncommitted changes, and confirmation that nothing was pushed.
