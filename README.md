# OmniScout Skills

Agent Skills for [OmniScout](https://omniscout.xyz) — local-first web search, research, extraction, browser and computer automation for AI agents.

## Repository structure

```
skills/
└── omniscout/
    ├── SKILL.md              # metadata and instructions the agent loads
    └── references/
        └── operations.md     # troubleshooting, loaded on demand
```

Each skill is a self-contained directory with a `SKILL.md` file. The `name` in its frontmatter matches the directory name, and supporting files are referenced from the skill root with relative paths.

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add omniscoutdotxyz/skills --skill omniscout -g -y
```

List the skills in this repository:

```bash
npx skills add omniscoutdotxyz/skills --list
```

Or let the OmniScout CLI install the skill and setup instructions:

```bash
pip install omniscout
omniscout install --skill
```

## Skills

| Skill | Description |
|-------|-------------|
| `omniscout` | Search, research, extract, browser memory, and full browser automation via `omniscout` |

## Creating a skill

A skill is a folder containing a `SKILL.md` file with YAML frontmatter and instructions:

```markdown
---
name: my-skill
description: What the skill does and when an agent should use it.
---

# My Skill

Instructions the agent follows when this skill is active.
```

`name` must be lowercase and match the folder name. `description` is limited to 1024 characters and should say what the skill does and when to use it. See the [specification](https://agentskills.io/specification) for the full set of fields.

## Validate

```bash
uvx --from skills-ref agentskills validate ./skills/omniscout
```

## Contributing

Improvements to the skill are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).
