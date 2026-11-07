> **Note:** This repository contains OmniScout's skills for AI agents. For information about the Agent Skills standard, see [agentskills.io](https://agentskills.io).

# Skills

Skills are folders of instructions and resources that an agent loads on demand to work better on a specific task. This repository packages the OmniScout skill in that format, so any agent that supports Agent Skills can search the web, research topics, extract page content, remember pages, and drive a real browser.

For more information, check out:

- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills)
- [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude)
- [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [Equipping agents for the real world with Agent Skills](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

# About This Repository

This repository contains the OmniScout skill. Search, research, extraction, and browser automation are documented in the skill itself, so it works the same whether an agent loads it from here or from a local install.

Each skill is self-contained in its own folder with a `SKILL.md` file containing the instructions and metadata the agent uses. Supporting references live next to it and are loaded only when needed.

# Skill Sets

- [./skills](./skills): The OmniScout skill
- [./template](./template): Skill template for new contributions

# Install

## Any Agent Skills client

```bash
npx skills add omniscoutdotxyz/skills --skill omniscout -g -y
```

List the skills in this repository:

```bash
npx skills add omniscoutdotxyz/skills --list
```

## Claude Code

Register this repository as a plugin marketplace:

```text
/plugin marketplace add omniscoutdotxyz/skills
```

Then install the skill:

```text
/plugin install omniscout@omniscout-skills
```

## OmniScout CLI

The OmniScout CLI can install this skill and its setup instructions for you:

```bash
pip install omniscout
omniscout install --skill
```

# Skills

| Skill | Description |
|-------|-------------|
| `omniscout` | Search, research, extract, browser memory, and full browser automation via `omniscout` |

# Creating a Basic Skill

A skill is just a folder with a `SKILL.md` file containing YAML frontmatter and instructions. Use the [template](./template/SKILL.md) as a starting point:

```markdown
---
name: my-skill-name
description: A clear description of what this skill does and when to use it
---

# My Skill Name

[Add your instructions here that the agent will follow when this skill is active]

## Examples
- Example usage 1
- Example usage 2

## Guidelines
- Guideline 1
- Guideline 2
```

The frontmatter requires only two fields:

- `name` — A unique identifier for your skill (lowercase, hyphens for spaces)
- `description` — A complete description of what the skill does and when to use it

The markdown content below contains the instructions, examples, and guidelines the agent follows while the skill is active.

# License

Modified MIT — see [LICENSE](LICENSE).
