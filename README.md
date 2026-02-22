# Wordmade Certification Plugin

**AI Certification skill for Claude Code** — integrate with the [Wordmade AI Certification](https://certification.wordmade.world) inverse CAPTCHA service.

## What This Plugin Provides

A Claude Code skill that enables AI agents to:

- **Solve certification challenges** — request, solve, and submit answers to the inverse CAPTCHA
- **Integrate the widget** — embed the certification widget on third-party sites
- **Manage sites** — register, configure, and manage certification sites via the customer portal API
- **Verify certificates** — check certificate validity and implement server-side verification

## Installation

### Claude Code Plugin (recommended)

```bash
# From GitHub release (stable URL, always latest)
claude /plugin install https://github.com/wordmade/cert-cc-plugin/releases/latest/download/certification-plugin-latest.zip

# Or load from local directory (for development)
claude --plugin-dir ./cert-cc-plugin
```

### Standalone Skill

If you just want the skill without the full plugin:

```bash
# Download SKILL.md directly from the API
mkdir -p ~/.claude/skills/certification
curl -o ~/.claude/skills/certification/SKILL.md https://certification.wordmade.world/skill.md
```

## Quick Start

Once installed, the certification skill is automatically available. Ask Claude Code to:

- "Solve a certification challenge at certification.wordmade.world"
- "Integrate the certification widget into my site"
- "Set up server-side certificate verification"

## Live Documentation

The certification service serves documentation directly — always up to date:

| URL | Content |
|-----|---------|
| [certification.wordmade.world/agents.md](https://certification.wordmade.world/agents.md) | Agent instructions (how to solve challenges) |
| [certification.wordmade.world/guide.md](https://certification.wordmade.world/guide.md) | Integration guide (widget, verification, security) |
| [certification.wordmade.world/api-reference.md](https://certification.wordmade.world/api-reference.md) | Full API reference |
| [certification.wordmade.world/skill.md](https://certification.wordmade.world/skill.md) | This skill (latest version) |
| [certification.wordmade.world/demo](https://certification.wordmade.world/demo) | Interactive widget customizer |

The SKILL.md in this plugin is a stable reference layer that points agents to these
live docs. Detailed documentation lives on the service itself and is always current.

## Plugin Contents

```
cert-cc-plugin/
├── .claude-plugin/plugin.json      # Claude Code plugin manifest
├── skills/certification/
│   └── SKILL.md                    # Certification skill (reference layer)
├── README.md                       # This file
└── LICENSE                         # MIT
```

## Building from Source

```bash
git clone https://github.com/wordmade/cert-cc-plugin.git
cd cert-cc-plugin

# Test locally
claude --plugin-dir .
```

## Related

- [Wordmade](https://wordmade.world) — The world for AI agents
- [Certification Portal](https://certification.wordmade.world) — Customer portal and documentation

## License

MIT
