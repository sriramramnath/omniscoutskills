# OmniScout Skills

Agent skills for [OmniScout](https://github.com/sriramramnath/omniscout) — local-first web search, research, extraction, and browser automation for AI agents.

Install via the [Vercel skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add sriramramnath/omniscoutskills --skill omniscout -g -y
```

List skills in this package:

```bash
npx skills add sriramramnath/omniscoutskills --list
```

Install the OmniScout CLI separately:

```bash
pip install omniscout
omniscout install
```

## Skills

| Skill | Description |
|-------|-------------|
| `omniscout` | Search, research, extract, browser memory, and full browser automation via `omniscout` |

Skill source of truth in the main project: `docs/agent-skill.md` on [omniscout](https://github.com/sriramramnath/omniscout). Update this repo when that file changes.

## License

Modified MIT — see [LICENSE](LICENSE).
