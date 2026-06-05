# Hawiyat n8n + Evolution API

Claude Code skill for designing, building, and managing n8n workflows and Evolution API WhatsApp automation on Hawiyat infrastructure.

## Quick Start

```bash
git clone github.com/Hawiyat-Org/hawiyat-n8n-skill
cd hawiyat-n8n-skill
claude .
```

The skill loads automatically. Provide your n8n instance URL and API key when prompted, then start building.

## What It Does

- **Workflow management** — create, update, activate/deactivate n8n workflows
- **WhatsApp chatbots** — build bots with memory, typing indicators, and anti-loop guards
- **Evolution API** — configure webhooks, send messages, manage instances
- **Best practices** — domain validation, node economy, anti-loop protection

## Prerequisites

- n8n instance on `*.hawiyat.cloud` or `*.hawiyat.org`
- n8n API key
- (Optional) Evolution API instance on a Hawiyat domain

## Installation Options

| Method | Command |
|--------|---------|
| **Open repo** (auto-load) | `claude .` inside the repo |
| **Global install** | `claude add skill ./hawiyat-n8n-evo.skill` |
| **Manual extract** | `unzip hawiyat-n8n-evo.skill -d .claude/skills/hawiyat-n8n-evo/` |

## Repository Structure

```
.
├── .claude/skills/hawiyat-n8n-evo/   # Skill definition & references
│   ├── SKILL.md                       # System instructions
│   └── references/
│       ├── n8n-workflow-schema.md
│       └── evolution-api-reference.md
├── hawiyat-n8n-evo.skill              # Distributable skill package
├── CLAUDE.md                          # Repo config (auto-loads skill)
└── README.md
```

## Development

Edit the skill files in `.claude/skills/hawiyat-n8n-evo/`, then rebuild the distributable ZIP:

```bash
cd .claude/skills && zip -r ../../hawiyat-n8n-evo.skill hawiyat-n8n-evo/
```

---

*Hawiyat — Proprietary. Use restricted to Hawiyat-authorized infrastructure.*
