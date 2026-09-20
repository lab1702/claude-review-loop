# claude-review-loop

A Claude Code plugin that adds the `review-loop` skill: iterative, whole-repository review by fresh, isolated reviewer subagents, with verified fixes committed locally until two consecutive passes are clean.

## Install

The repository doubles as its own plugin marketplace. In Claude Code:

```text
/plugin marketplace add lab1702/claude-review-loop
/plugin install review-loop@claude-review-loop
```

The repository is private, so the machine running Claude Code needs GitHub credentials that can read it (for example `gh auth login`).

## Use

Invoke the skill explicitly, and authorize local commits on the current branch in the same request, for example:

```text
/review-loop I authorize ordinary commits to the current branch.
```

The skill never pushes. It requires a clean working tree on a checked-out branch and host-provided isolated subagents. See [skills/review-loop/SKILL.md](skills/review-loop/SKILL.md) for the full procedure, run boundaries, and the final report format.

## Layout

```text
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   self-hosted marketplace catalog
skills/review-loop/SKILL.md       the skill
```
