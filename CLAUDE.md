# Hawiyat n8n + Evolution API — Claude Code Skill

This repo auto-loads the `hawiyat-n8n-evo` skill when opened in Claude Code.

## ⚠️ Critical Rule: Credentials Before Code

**The skill MUST collect and validate credentials (n8n instance URL, API keys) BEFORE designing or outputting any workflow JSON, API calls, or architecture.** This is enforced in `SKILL.md` under the SESSION SETUP — HARD GATE section. Do not skip it.

## Install Globally (Always Available)

To install the skill globally so it loads in every Claude Code session — not just when inside this repo — run:

```bash
claude add skill ./hawiyat-n8n-evo.skill
```

After that, the skill is available from any directory without needing to `cd` into this repo.

See [README.md](README.md) for full details.
