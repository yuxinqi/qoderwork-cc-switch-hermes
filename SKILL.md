# CC Switch + Hermes 配置管理

当用户需要配置、排查或修复 CC Switch 与 Hermes Agent 的模型供应商配置时使用此技能。

## 架构概览

CC Switch 是一个跨平台桌面应用（GitHub: farion1231/cc-switch），用于统一管理 AI 编程工具（Claude Code、Codex、Hermes、Gemini CLI 等）的 API 供应商配置。它的核心数据存储和配置联动机制如下：

- **CC Switch 数据库**：`~/.cc-switch/cc-switch.db`（SQLite），存储所有供应商配置
- **Hermes 配置文件**：`~/.hermes/config.yaml`，Hermes Agent 实际读取的运行配置
- **关键联动点**：CC Switch 切换供应商时会写入 config.yaml 的 `model.default`、`model.provider` 和 `custom_providers`

CC Switch 和 Hermes 共享同一个 config.yaml，在 CC Switch 中删除或修改供应商会直接影响 Hermes 运行。

## 数据库结构（providers 表）

```sql
CREATE TABLE providers (
    id TEXT NOT NULL,
    app_type TEXT NOT NULL,         -- 'hermes', 'claude', 'codex', 'openclaw' 等
    name TEXT NOT NULL,
    settings_config TEXT NOT NULL,  -- JSON 字符串，核心配置
    website_url TEXT,
    category TEXT,
    created_at INTEGER,
    sort_index INTEGER,
    meta TEXT NOT NULL DEFAULT '{}', -- JSON，存放 testConfig 等元数据
    is_current BOOLEAN NOT NULL DEFAULT 0,
    ...
    PRIMARY KEY (id, app_type)
);
```

### settings_config 的 Hermes 格式（最关键的坑点）

**正确格式**（对象数组）：

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

**错误格式（常见坑）**：

```json
{
  "models": ["deepseek-v4-flash", "sensenova-6.7-flash-lite"]
}
```

简单字符串数组会导致 CC Switch 无法提取 `models[0].id` 作为默认模型，切换供应商时 config.yaml 的 `model.default` 不会更新，保留旧值（通常是 `anthropic/claude-opus-4-8`）。

### models 字段的作用链

```
CC Switch DB settings_config.models[0].id
    ↓ (切换供应商时)
config.yaml model.default
    ↓ (Hermes 运行时)
辅助任务 (title_generation 等) 使用的模型
```

### api_mode 字段

`api_mode` 指定 API 协议类型，从 v3.14.0 开始必须显式设置（移除了 Auto 模式）：

| 值 | 协议 | 适用场景 |
|---|---|---|
| `chat_completions` | OpenAI 兼容 | 大多数中转、OpenRouter、DeepSeek 等 |
| `anthropic_messages` | Anthropic 原生 | Anthropic 官方端点 |
| `codex_responses` | OpenAI Responses API | OpenAI 新版端点 |
| `bedrock_converse` | AWS Bedrock | 健康检查不支持 |

缺少 `api_mode` 会导致 stream check 报错："Hermes 供应商缺少 api_mode 字段"。

### meta 字段（测试模型配置）

CC Switch 对 Hermes 供应商的默认健康检查测试模型是硬编码的 `gpt-4o`。如果供应商不提供 `gpt-4o`（比如 sensenova、agnes-ai），健康检查会返回 404。通过 `meta.testConfig` 覆盖：

```json
{
  "testConfig": {
    "enabled": true,
    "test_model": "deepseek-v4-flash"
  }
}
```

## Hermes config.yaml 结构要点

### 顶层模型配置

```yaml
model:
  default: agnes-2.0-flash        # 当前默认模型 ID
  provider: agnes-ai-hermes       # 当前供应商 name
```

CC Switch 切换供应商时，`default` 取自 `settings_config.models[0].id`，`provider` 取自 `settings_config.name`。

### custom_providers（Hermes 专有供应商）

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

注意 config.yaml 中 `models` 是 YAML dict 格式（键为模型 ID），与数据库中的对象数组格式不同。CC Switch 写入时会自动转换。

### auxiliary 辅助任务

```yaml
auxiliary:
  title_generation:
    provider: auto    # 推荐用 auto，跟随主供应商
    model: ''
  compression:
    provider: auto
    model: google/gemini-3-flash-preview
  # ... 其他类似
```

**推荐 `provider: auto`**。如果硬编码为某个供应商，当用户切换主供应商后，辅助任务仍用旧供应商的模型，但 `model.default` 已变，会导致 404。

## 踩坑记录与排查指南

### 坑1：models 格式不对导致默认模型不更新

**现象**：CC Switch 模型列表显示 `anthropic/claude-opus-4-8`，而非供应商自己的模型。
**原因**：数据库中 `settings_config.models` 是字符串数组 `["model-id"]` 而非对象数组 `[{"id": "model-id", "name": "Name"}]`。CC Switch 从 `models[0].id` 取值，字符串没有 `.id` 属性，保留了旧值。
**修复**：将所有 models 转为 `[{"id": "xxx", "name": "Xxx"}]` 格式。

### 坑2：缺少 api_mode 导致健康检查失败

**现象**："Hermes 供应商缺少 api_mode 字段"
**原因**：v3.14.0 之前创建的供应商或旧版 deeplink 导入的配置没有 `api_mode` 字段。
**修复**：在 `settings_config` 中添加 `"api_mode": "chat_completions"`。

### 坑3：env 格式混入 Hermes 配置

**现象**：Hermes 供应商的 settings_config 是 `{"env": {"ANTHROPIC_AUTH_TOKEN": "..."}}` 格式。
**原因**：这是 Claude Code 的配置格式，被错误地复制到了 Hermes 供应商。
**修复**：重写为 Hermes 标准格式（`base_url`、`api_key`、`api_mode`、`models`）。

### 坑4：辅助任务 provider 硬编码导致切换后 404

**现象**："Auxiliary title generation failed: HTTP 404: model is not found"
**原因**：辅助任务的 `provider` 硬编码为某个供应商（如 sensenova），但主模型已切换到其他供应商的模型（如 agnes-2.0-flash），目标供应商不提供该模型。
**修复**：所有辅助任务用 `provider: auto`，让它们跟随当前主供应商。

### 坑5：测试模型 gpt-4o 不可用

**现象**："sensenova 测试模型 gpt-4o 不存在或已下架"
**原因**：CC Switch 对 Hermes 的默认测试模型是硬编码的 `gpt-4o`，但许多供应商不提供此模型。
**修复**：在 `meta` 中设置 `testConfig.test_model` 为供应商实际支持的模型。

### 坑6：在 CC Switch 删除供应商连带影响 Hermes

**现象**：删除 CC Switch 供应商后，Hermes 报 "Primary auth failed — switching to fallback"。
**原因**：CC Switch 和 Hermes 共享 config.yaml，删除操作会清空 `custom_providers`。
**修复**：操作前备份 config.yaml，删除后检查并恢复必要的 custom_providers。

## 常用操作命令

### 查询所有 Hermes 供应商

```bash
sqlite3 ~/.cc-switch/cc-switch.db \
  "SELECT id, name, json_extract(settings_config, '$.api_mode') as api_mode, \
   json_extract(settings_config, '$.base_url') as base_url \
   FROM providers WHERE app_type = 'hermes';"
```

### 检查 models 格式是否正确

```bash
python3 -c "
import sqlite3, json
conn = sqlite3.connect('$HOME/.cc-switch/cc-switch.db')
for id, name, cfg in conn.execute(
    \"SELECT id, name, settings_config FROM providers WHERE app_type = 'hermes'\"\"):
    models = json.loads(cfg).get('models', [])
    ok = models and isinstance(models[0], dict) and 'id' in models[0]
    print(f\"{'OK' if ok else 'BAD'} {name}: {models[0] if models else 'empty'}\")
"
```

### 批量修复 models 格式

```python
import sqlite3, json
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

### 备份配置

```bash
cp ~/.cc-switch/cc-switch.db ~/.cc-switch/cc-switch.db.bak
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak
```

## 相关资源

- GitHub: https://github.com/farion1231/cc-switch
- 源码关键路径：
  - `src-tauri/src/hermes_config.rs` — Hermes 配置读写、models 转换
  - `src-tauri/src/services/stream_check.rs` — 健康检查、api_mode 校验
  - `src/components/providers/forms/HermesFormFields.tsx` — 前端表单
  - `src/config/hermesProviderPresets.ts` — 预设、HermesApiMode 类型定义
