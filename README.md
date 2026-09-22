# claude-review-loop

A Claude Code plugin that adds the `review-loop` skill: iterative, whole-repository review by fresh, isolated reviewer subagents, with verified fixes committed locally until two consecutive passes are clean.

## Install

The repository doubles as its own plugin marketplace. In Claude Code, run each command separately (the `/plugin` command accepts only a single line):

```text
/plugin marketplace add lab1702/claude-review-loop
```

```text
/plugin install review-loop@claude-review-loop
```

## Use

Invoke the skill explicitly with the `/review-loop:review-loop` slash command (Claude Code prefixes plugin skills with the plugin name), and authorize local commits on the current branch in the same request, for example:

```text
/review-loop:review-loop I authorize ordinary commits to the current branch.
```

Claude never runs this skill on its own; only the slash command starts it.

Each pass:

1. A fresh, read-only reviewer subagent, with no access to earlier conversation or findings, reviews the whole repository at the current commit.
2. Claude verifies each finding, fixes the real ones, and runs the project's checks, reusing earlier results when the content has not changed.
3. Verified fixes are committed locally to the current branch.

The run finishes when two passes in a row find nothing on the same commit, or stops as blocked after 10 passes or when something needs your attention. It ends with a report of fixes, checks, commits, and any remaining limitations.

Requirements and guarantees:

- A clean working tree on a checked-out branch, with no merge, rebase, or similar operation in progress.
- Dependency installs and checks should leave tracked files unchanged (for example, an up-to-date lockfile). Claude deletes stray untracked files that checks leave behind and lists them in the report so you can add ignore rules; changes to tracked files stop the run unless they fall within a verified fix (for example, a formatter reformatting fixed code).
- Configured commit hooks run as part of the checks before each commit, through the hook framework's own command or `git hook run`; a hook that still rejects the commit stops the run.
- A check that fails without changing files gets one unchanged rerun per pass; if it then passes, it is reported as flaky instead of stopping the run.
- No pushes, branch switches, amends, or history rewrites. Pushing is left to you.

Repository size limit: each pass uses a single reviewer for the whole repository, and there is no option to review only part of it. If the repository is too large for one reviewer to cover, the reviewer reports a coverage gap, the review is rejected, and the run stops as blocked instead of completing. Generated and vendored content does not need detailed inspection when its inputs and integration are reviewed, so the limit depends mainly on the size of the maintained code.

Trust and permissions: run the skill only on repositories you trust. Claude installs dependencies and runs the project's tests, builds, and hooks with network access, and reviewers read all repository content, which could include instructions aimed at an AI agent. A run also makes many edits and runs many shell commands, so expect frequent approval prompts in the default permission mode. To reduce them, allow the project's check commands and local Git commands in your permission settings, but not `git push`. Bypass permissions only in an isolated environment.

See [skills/review-loop/SKILL.md](skills/review-loop/SKILL.md) for the full procedure, run boundaries, and the final report format.

## Layout

```text
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   self-hosted marketplace catalog
skills/review-loop/SKILL.md       the skill
agents/reviewer.md                read-only reviewer subagent
```

## License

[MIT](LICENSE)
