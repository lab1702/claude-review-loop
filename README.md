# claude-review-loop

A Claude Code plugin that adds the `review-loop` skill: iterative, whole-repository review by fresh, isolated reviewer subagents, with verified fixes committed locally until two consecutive passes are clean.

## Install

The repository doubles as its own plugin marketplace. In Claude Code:

Run each command separately (the `/plugin` command only accepts a single line):

```text
/plugin marketplace add lab1702/claude-review-loop
```

```text
/plugin install review-loop@claude-review-loop
```

## Use

Invoke the skill explicitly with the `/review-loop` slash command, and authorize local commits on the current branch in the same request, for example:

```text
/review-loop I authorize ordinary commits to the current branch.
```

Claude never runs this skill on its own; only the slash command starts it. The skill never pushes. It requires a clean working tree on a checked-out branch and host-provided isolated subagents. See [skills/review-loop/SKILL.md](skills/review-loop/SKILL.md) for the full procedure, run boundaries, and the final report format.

## Layout

```text
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   self-hosted marketplace catalog
skills/review-loop/SKILL.md       the skill
```

## License

[MIT](LICENSE)
