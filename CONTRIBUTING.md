# Contributing

Contributions are welcome when they make the OmniScout skill better for the agents that use it.

## What to contribute

Open a pull request when you have something concrete:

- A step, command, or order of steps that works better in practice.
- A failure mode or edge case the skill does not handle yet.
- A correction to instructions that send an agent down the wrong path.
- A reference file that keeps `SKILL.md` focused.

Good contributions come from real use. If you hit something while working with OmniScout and the skill pointed you the wrong way, that is exactly the kind of change we want.

## What not to contribute

- General documentation, tutorials, or marketing copy.
- Changes to the OmniScout CLI itself. This repository only contains the skill.
- Speculative rewrites or style-only edits.
- New skills without a real use case.

## Before you open a PR

1. Keep the change small and scoped to one improvement.
2. Keep `SKILL.md` under 500 lines; move detail into `references/`.
3. Make sure the folder name and the `name` in the frontmatter match.
4. Run the checks:

```bash
agentskills validate ./skills/omniscout
npx skills add . --skill omniscout --agent claude-code --copy -y
```

## Pull request

Describe what you were doing, what the skill told you, and what should happen instead. A before/after example or a transcript is more useful than a summary.

All pull requests are reviewed before merge. Contributions that do not improve how an agent uses OmniScout may be closed.
