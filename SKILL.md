# CC Switch + Hermes Provider Configuration

Use this skill when the user needs to configure, debug, or fix CC Switch and Hermes Agent model provider settings.

## Architecture Overview

CC Switch is a cross-platform desktop application ([GitHub: farion1231/cc-switch](https://github.com/farion1231/cc-switch)) that unifies API provider management for AI coding tools (Claude Code, Codex, Hermes, Gemini CLI, OpenCode, etc.). Key data stores:

- **CC Switch database**: `~/.cc-switch/cc-switch.db` (SQLite) — stores all provider configurations
- **Hermes config file**: `~/.hermes/config.yaml` — runtime config read by Hermes Agent
- **Critical coupling**: When CC Switch switches a provider, it writes `model.default`, `model.provider`, and `custom_providers` into config.yaml

CC Switch and Hermes share the same config.yaml. Deleting or modifying providers in CC Switch directly affects Hermes runtime behavior.

## Database Schema (providers table)

```sql
CREATE TABLE providers (
    id TEXT NOT NULL,
    app_type TEXT NOT NULL,         -- 'hermes', 'claude', 'codex', 'openclaw', etc.
    name TEXT NOT NULL,
    settings_config TEXT NOT NULL,  -- JSON string — core configuration
    website_url TEXT,
    category TEXT,
    created_at INTEGER,
    sort_index INTEGER,
    meta TEXT NOT NULL DEFAULT '{}', -- JSON — stores testConfig and other metadata
    is_current BOOLEAN NOT NULL DEFAULT 0,
    ...
    PRIMARY KEY (id, app_type)
);
```

### settings_config Format for Hermes (Most Critical Pitfall)

**Correct format** (array of objects):

```json
{
  "name": "sensenova",
  "base_url": "https://token.sensenova.cn/v1",
  "api_key": "sk-xxx",
  "api_mode": "chat_completions",
  "models": [
    {"id": "deepseek-v4-flash", "name": "DeepSeek V4 Flash"},
    {"id": "sensenova-6.7-flash-lite", "name": "SenseNova 6.7 Flash Lite"}
  ]
}
```

**Wrong format (common pitfall)**:

```json
{
  "models": ["deepseek-v4-flash", "sensenova-6.7-flash-lite"]
}
```

A plain string array prevents CC Switch from extracting `models[0].id` as the default model. When switching providers, `model.default` in config.yaml won't update and retains the previous value (typically `anthropic/claude-opus-4-8`).

### The models Field Data Flow

```
CC Switch DB: settings_config.models[0].id
    ↓ (on provider switch)
config.yaml: model.default
    ↓ (at Hermes runtime)
Auxiliary tasks (title_generation, etc.) use this model
```

### api_mode Field

`api_mode` specifies the API protocol type. Required since v3.14.0 (Auto mode was removed):

| Value | Protocol | Use Case |
|---|---|---|
| `chat_completions` | OpenAI-compatible | Most proxies, OpenRouter, DeepSeek, etc. |
| `anthropic_messages` | Anthropic native | Anthropic official endpoints |
| `codex_responses` | OpenAI Responses API | OpenAI newer endpoints |
| `bedrock_converse` | AWS Bedrock | Health check not supported |

Missing `api_mode` causes stream check error: "Hermes provider is missing the 'api_mode' field".

### meta Field (Test Model Configuration)

CC Switch hardcodes `gpt-4o` as the default health check test model for Hermes providers. If a provider doesn't serve `gpt-4o` (e.g., sensenova, agnes-ai), health checks return 404. Override via `meta.testConfig`:

```json
{
  "testConfig": {
    "enabled": true,
    "test_model": "deepseek-v4-flash"
  }
}
```

## Hermes config.yaml Key Sections

### Top-Level Model Configuration

```yaml
model:
  default: agnes-2.0-flash        # Current default model ID
  provider: agnes-ai-hermes       # Current provider name
```

When CC Switch switches providers: `default` is taken from `settings_config.models[0].id`, `provider` from `settings_config.name`.

### custom_providers (Hermes-Specific Providers)

```yaml
custom_providers:
- name: sensenova
  base_url: https://token.sensenova.cn/v1
  api_key: sk-xxx
  api_mode: chat_completions
  models:
    deepseek-v4-flash: {}
    sensenova-6.7-flash-lite: {}
- name: agnes-ai-hermes
  base_url: https://apihub.agnes-ai.com/v1
  api_key: sk-xxx
  api_mode: chat_completions
  models:
    agnes-2.0-flash: {}
```

Note: In config.yaml, `models` uses YAML dict format (keyed by model ID), different from the database's object array format. CC Switch auto-converts when writing.

### Auxiliary Tasks

```yaml
auxiliary:
  title_generation:
    provider: auto    # Recommended: follows main provider
    model: ''
  compression:
    provider: auto
    model: google/gemini-3-flash-preview
  # ... others follow the same pattern
```

**Recommended: `provider: auto`**. Hardcoding a specific provider causes 404 errors after switching the main provider — the auxiliary task still uses the old provider's model, but `model.default` has changed.

## Pitfalls & Troubleshooting Guide

### Pitfall 1: Wrong models format — default model doesn't update

**Symptom**: CC Switch model list shows `anthropic/claude-opus-4-8` instead of the provider's own models.
**Cause**: `settings_config.models` is a string array `["model-id"]` instead of an object array `[{"id": "model-id", "name": "Name"}]`. CC Switch reads `models[0].id`; strings don't have `.id`, so the old value persists.
**Fix**: Convert all models to `[{"id": "xxx", "name": "Xxx"}]` format.

### Pitfall 2: Missing api_mode — health check fails

**Symptom**: "Hermes provider is missing the 'api_mode' field"
**Cause**: Providers created before v3.14.0 or imported via legacy deeplinks lack the `api_mode` field.
**Fix**: Add `"api_mode": "chat_completions"` to `settings_config`.

### Pitfall 3: Claude Code env format in Hermes config

**Symptom**: Hermes provider's settings_config is `{"env": {"ANTHROPIC_AUTH_TOKEN": "..."}}` format.
**Cause**: Claude Code's configuration format was mistakenly copied to a Hermes provider.
**Fix**: Rewrite to standard Hermes format (`base_url`, `api_key`, `api_mode`, `models`).

### Pitfall 4: Hardcoded auxiliary provider causes 404 after switch

**Symptom**: "Auxiliary title generation failed: HTTP 404: model is not found"
**Cause**: An auxiliary task's `provider` was hardcoded to a specific provider (e.g., sensenova), but the main model switched to a different provider's model (e.g., agnes-2.0-flash) that the hardcoded provider doesn't serve.
**Fix**: Set all auxiliary tasks to `provider: auto` so they follow the current main provider.

### Pitfall 5: Default test model gpt-4o unavailable

**Symptom**: "Test model gpt-4o does not exist or has been delisted"
**Cause**: CC Switch hardcodes `gpt-4o` as the default test model for Hermes, but many providers don't serve it.
**Fix**: Set `meta.testConfig.test_model` to a model the provider actually supports.

### Pitfall 6: Deleting providers in CC Switch cascades to Hermes

**Symptom**: After deleting a CC Switch provider, Hermes reports "Primary auth failed — switching to fallback".
**Cause**: CC Switch and Hermes share config.yaml. Deletion clears `custom_providers`.
**Fix**: Back up config.yaml before operations; verify and restore custom_providers after deletion.

## Common Commands

### Query all Hermes providers

```bash
sqlite3 ~/.cc-switch/cc-switch.db \
  "SELECT id, name, json_extract(settings_config, '$.api_mode') as api_mode, \
   json_extract(settings_config, '$.base_url') as base_url \
   FROM providers WHERE app_type = 'hermes';"
```

### Validate models format

```bash
python3 -c "
import sqlite3, json
conn = sqlite3.connect('$HOME/.cc-switch/cc-switch.db')
for id, name, cfg in conn.execute(
    \"SELECT id, name, settings_config FROM providers WHERE app_type = 'hermes'\"):
    models = json.loads(cfg).get('models', [])
    ok = models and isinstance(models[0], dict) and 'id' in models[0]
    print(f\"{'OK' if ok else 'BAD'} {name}: {models[0] if models else 'empty'}\")
"
```

### Batch-fix models format

```python
import sqlite3, json, os
conn = sqlite3.connect(os.path.expanduser("~/.cc-switch/cc-switch.db"))
for pid, pname, cfg_str in conn.execute(
    "SELECT id, name, settings_config FROM providers WHERE app_type = 'hermes'"):
    cfg = json.loads(cfg_str)
    models = cfg.get("models", [])
    if models and isinstance(models[0], str):
        cfg["models"] = [{"id": m, "name": m.split("/")[-1].replace("-"," ").title()} for m in models]
        conn.execute("UPDATE providers SET settings_config = ? WHERE id = ? AND app_type = 'hermes'",
                     (json.dumps(cfg), pid))
        print(f"Fixed: {pname}")
conn.commit()
```

### Backup before changes

```bash
cp ~/.cc-switch/cc-switch.db ~/.cc-switch/cc-switch.db.bak
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
```

## Related Resources

- GitHub: https://github.com/farion1231/cc-switch
- Key source paths:
  - `src-tauri/src/hermes_config.rs` — Hermes config read/write, models conversion
  - `src-tauri/src/services/stream_check.rs` — Health check, api_mode validation
  - `src/components/providers/forms/HermesFormFields.tsx` — Frontend form
  - `src/config/hermesProviderPresets.ts` — Presets, HermesApiMode type definition
