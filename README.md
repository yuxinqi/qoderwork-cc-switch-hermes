# CC Switch + Hermes Configuration Skill for QoderWork

A QoderWork skill that encodes hard-won lessons from configuring [CC Switch](https://github.com/farion1231/cc-switch) to manage model providers for [Hermes Agent](https://github.com/NousResearch/hermes-agent). It covers database internals, config file structures, common failure modes, and proven fix procedures.

## What Is This?

This repository contains a single [SKILL.md](./SKILL.md) file — a structured knowledge document designed to be installed as a skill in [QoderWork](https://qoder.com). When installed, QoderWork automatically loads this knowledge whenever you ask it to configure, debug, or fix CC Switch and Hermes provider settings.

## Background

CC Switch is a cross-platform desktop app (8k+ GitHub stars) that unifies API provider management for AI coding tools — Claude Code, Codex, Hermes, Gemini CLI, OpenCode, and more. It stores configurations in a local SQLite database (`~/.cc-switch/cc-switch.db`) and writes runtime configs to each tool's native config file (e.g., `~/.hermes/config.yaml` for Hermes).

The interaction between CC Switch's database and Hermes's config file is subtle and poorly documented. This skill captures the exact data formats, field semantics, and failure patterns discovered through real debugging sessions.

## What It Covers

### Architecture & Data Flow
- CC Switch SQLite database schema (`providers` table)
- Hermes `config.yaml` structure and how CC Switch writes to it
- The shared `config.yaml` coupling between CC Switch and Hermes

### Critical Data Formats
- **`settings_config.models`**: Must be an array of objects `[{"id": "...", "name": "..."}]`, not plain strings — the #1 source of bugs
- **`api_mode`**: Required since CC Switch v3.14.0; four valid values mapped to API protocols
- **`meta.testConfig`**: Per-provider health check model override (default `gpt-4o` doesn't work for most providers)

### Six Documented Pitfalls
1. **Wrong models format** — String arrays silently fail to update `model.default` on provider switch
2. **Missing `api_mode`** — Pre-v3.14.0 configs lack this field; health check fails with a cryptic error
3. **Claude Code `env` format in Hermes** — Copy-pasting Claude Code config into a Hermes provider breaks everything
4. **Hardcoded auxiliary providers** — Binding `title_generation` or `compression` to a specific provider causes 404s after switching
5. **Default test model `gpt-4o`** — Most third-party providers don't serve `gpt-4o`; health checks fail with 404
6. **Cascading deletes** — Removing a provider in CC Switch wipes it from Hermes's `custom_providers` too

### Ready-to-Use Commands
- Query all Hermes providers and their config status
- Validate `models` format across all providers
- Batch-fix string arrays to object arrays
- Backup procedures for both database and config files

## Installation

### As a QoderWork Skill
1. Clone this repo or download `SKILL.md`
2. Place it in your QoderWork skills directory: `~/.qoderwork/skills/cc-switch-hermes/SKILL.md`
3. QoderWork will auto-discover it on next use

### As a Reference
Simply read [SKILL.md](./SKILL.md) — it's a standalone Markdown document.

## Quick Start: Common Operations

```bash
# List all Hermes providers with key fields
sqlite3 ~/.cc-switch/cc-switch.db \
  "SELECT id, name, json_extract(settings_config, '$.api_mode') as api_mode,
   json_extract(settings_config, '$.base_url') as base_url
   FROM providers WHERE app_type = 'hermes';"

# Check if models format is correct (object array vs string array)
python3 -c "
import sqlite3, json
conn = sqlite3.connect('$HOME/.cc-switch/cc-switch.db')
for pid, name, cfg in conn.execute(
    \"SELECT id, name, settings_config FROM providers WHERE app_type = 'hermes'\"):
    models = json.loads(cfg).get('models', [])
    ok = models and isinstance(models[0], dict) and 'id' in models[0]
    print(f\"{'OK' if ok else 'BAD'} {name}: {models[0] if models else 'empty'}\")"

# Backup before any changes
cp ~/.cc-switch/cc-switch.db ~/.cc-switch/cc-switch.db.bak
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
```

## Key Source Paths in CC Switch

| File | Purpose |
|---|---|
| `src-tauri/src/hermes_config.rs` | Hermes config read/write, models array-to-dict conversion |
| `src-tauri/src/services/stream_check.rs` | Health check logic, `api_mode` validation, test model resolution |
| `src/components/providers/forms/HermesFormFields.tsx` | Frontend provider form, default model derivation |
| `src/config/hermesProviderPresets.ts` | Provider presets, `HermesApiMode` type definition |

## License

MIT
