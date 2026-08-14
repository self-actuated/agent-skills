# Self-Actuated Agent Skills

Agent Skills for the self-hosted actuated GitHub Actions runner platform.

## Available Skills

### actuated-cli

`skills/actuated-cli`

Diagnose and repair actuated CI runner dispatch failures — stuck queued jobs,
dead hosts, unreachable runners. Covers GitHub status checks, queue and runner
inspection, agent/VM/controller logs, upgrades, and repairs.

## Installation

```bash
npx skills add self-actuated/agent-skills
```

### Claude Code Plugin

```bash
/plugin marketplace add self-actuated/agent-skills
/plugin install actuated-cli@self-actuated   # or the configured plugin name
```

### Manual

```bash
git clone https://github.com/self-actuated/agent-skills.git
cp -r agent-skills/skills/actuated-cli ~/.claude/skills/   # Claude Code
cp -r agent-skills/skills/actuated-cli ~/.agents/skills/   # Amp / Codex
cp -r agent-skills/skills/actuated-cli ~/.cursor/skills/   # Cursor
```

## License

MIT — see [LICENSE](LICENSE).
