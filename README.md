# Codex × opencode go (deepseek-v4-flash) + mimo-v2.5 识图 — 部署方案

在新电脑上快速配置 Codex 使用 **opencode go 订阅的 deepseek-v4-flash** 作为主模型，并附带 **mimo-v2.5 多模态识图**能力（图片任务自动用 mimo-v2.5 描述，替代主模型的视觉盲区）。

---

## 一、架构总览

```
Codex 桌面版 / CLI
  │  config.toml: model_provider="opencode" → https://opencode.ai/zen/go/v1
  │  wire_api = "responses"（原生协议，无需任何中间桥接）
  ▼
opencode.ai/zen/go/v1  (opencode go 订阅，Responses API)
  ├── deepseek-v4-flash  主模型（文本，不支持图像）
  └── mimo-v2.5          识图模型（多模态 text+image）

图片任务流程:
用户发图片 → codex 检测主模型不支持图像 → 替换为占位符
→ 主模型按 developer_instructions + describe-image 技能
→ 运行 describe_image.py → mimo-v2.5 识图 → 文字描述 → 回答
```

**设计要点**：
- **零中间件**：opencode.ai 原生支持 Responses API（apply_patch / web_search / reasoning 全部原生），不需要任何 bridge/代理
- **视觉回退**：主模型明确 `['text']` 模态（不能看图）→ 图片任务自动走 mimo-v2.5
- **子代理**：`[agents] default_subagent_model = "mimo-v2.5"`，复杂图像任务可 spawn mimo-v2.5 子代理
- **防 OCR 误用**：describe-image 技能引导模型走 mimo-v2.5；移走 layout-ocr/smart-ocr 技能避免模型自己折腾 OCR

---

## 二、快速开始（一键脚本，推荐）

### 前提

1. 已安装 Codex（CLI 或 ChatGPT 桌面版），且**至少启动过一次**（生成 `~/.codex` 目录）
2. 有 opencode go 订阅的 **API Key**（`sk-` 开头）

### 执行

**方式一：curl 一键安装（推荐，从 GitHub）**

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/BitQAI/codex-opencode-setup/main/setup-codex-opencode.sh)
```

**方式二：本地脚本**

```bash
bash ~/.codex/opencode-codex-setup/setup-codex-opencode.sh
# 或从本目录直接运行
cd ~/.codex/opencode-codex-setup && bash setup-codex-opencode.sh
```

脚本会：
1. 备份现有 `config.toml` / `models.json` / OCR 技能（到 `~/.codex/backup-opencode-codex/`）
2. 写入 `models.json`（deepseek-v4-flash + mimo-v2.5 完整条目）
3. 修改 `config.toml`（provider / agents / 识图指引，**保留 MCP、项目信任等现有配置**）
4. 创建识图脚本 `~/.codex/opencode-bridge-go/describe_image.py`
5. 创建 `describe-image` 技能（`~/.agents/skills/` + `~/.codex/skills/`）
6. 移走 `layout-ocr` / `smart-ocr` 技能（备份，避免模型误用 OCR）
7. 验证 TOML/JSON 语法

### 完成

**重启 Codex** 后即可使用。测试：
- 普通对话（deepseek-v4-flash 直连）
- 发一张图片 → 应自动用 mimo-v2.5 描述
- 让模型用 `apply_patch` 改文件 → 应原生成功

---

## 三、手动配置（备选，不跑脚本时）

### 1. `~/.codex/models.json`

写入两个模型的完整元数据（内容与脚本内嵌模板一致，见 `setup-codex-opencode.sh` 内 `MODELS_JSON_EOF` 段）：
- `deepseek-v4-flash`：`input_modalities: ["text"]`（**关键**：明确不能看图，触发视觉回退）
- `mimo-v2.5`：`input_modalities: ["text", "image"]`（识图模型）

### 2. `~/.codex/config.toml`

```toml
model = "deepseek-v4-flash"
model_provider = "opencode"
model_reasoning_effort = "high"
model_catalog_json = "~/.codex/models.json"

developer_instructions = "IMAGE HANDLING (MANDATORY): 遇到图片时运行 python3 ~/.codex/opencode-bridge-go/describe_image.py <path> [question] 获取 mimo-v2.5 的文字描述，禁止自己 OCR..."

[model_providers.opencode]
name = "opencode go"
base_url = "https://opencode.ai/zen/go/v1"
wire_api = "responses"
experimental_bearer_token = "<你的 opencode API Key>"

[agents]
default_subagent_model = "mimo-v2.5"
default_subagent_reasoning_effort = "medium"
```

### 3. 识图脚本 `~/.codex/opencode-bridge-go/describe_image.py`

内容见 `setup-codex-opencode.sh` 内 `DESCRIBE_EOF` 段（或从本机现有文件复制）。

```bash
python3 ~/.codex/opencode-bridge-go/describe_image.py <图片路径或URL> [问题]
# 输出: {"description": "..."}
```

### 4. 识图技能 `~/.agents/skills/describe-image/SKILL.md`（及 `~/.codex/skills/describe-image/`）

内容见脚本内 `SKILL_EOF` 段。它让模型在图片任务时优先看到"用 mimo-v2.5 识图"的方案。

### 5. 移走 OCR 技能（可选但推荐）

```bash
mkdir -p ~/.codex/backup-opencode-codex/disabled-skills-ocr
mv ~/.agents/skills/layout-ocr ~/.codex/backup-opencode-codex/disabled-skills-ocr/
mv ~/.agents/skills/smart-ocr ~/.codex/backup-opencode-codex/disabled-skills-ocr/
```

否则模型看到 OCR 技能会倾向自己 OCR（效果差），而不是用 mimo-v2.5。

---

## 四、识图功能机制

| 环节 | 机制 |
|---|---|
| 主模型不能看图 | `deepseek-v4-flash` 的 `input_modalities: ["text"]` → codex 把图片替换为占位符 |
| 模型知道怎么办 | `developer_instructions`（developer 消息）+ `describe-image` 技能（system 注入） |
| 实际识图 | `describe_image.py` 调 opencode 的 **mimo-v2.5** → 返回文字描述 |
| 复杂图像任务 | spawn **mimo-v2.5 子代理**（`[agents] default_subagent_model`） |
| 搜索 | 原生 `web_search`（opencode 服务端执行） |
| 文件编辑 | 原生 `apply_patch`（custom_tool_call） |

---

## 五、验证

```bash
# 1. 配置语法
python3 -c "import tomllib; tomllib.load(open('$HOME/.codex/config.toml','rb')); print('config OK')"
python3 -c "import json; json.load(open('$HOME/.codex/models.json')); print('models OK')"

# 2. 识图脚本
python3 ~/.codex/opencode-bridge-go/describe_image.py /path/to/image.png

# 3. 直连 opencode（可选，确认 key 有效）
curl -s -X POST "https://opencode.ai/zen/go/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <key>" \
  -d '{"model":"deepseek-v4-flash","input":"hi","stream":false}'
```

---

## 六、故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| 重启后模型还是自己 OCR | 技能未注入 / codex 未重启 | 确认重启 Codex；检查 `~/.agents/skills/describe-image/` 存在 |
| 图片路径找不到 | 剪贴板图片在 `/var/folders` 临时目录 | developer_instructions 已提示；模型用 `find` 定位 |
| apply_patch 报错 | 直连后应原生支持 | 确认 `wire_api="responses"`（非 bridge 地址） |
| 多轮后 reasoning 400 | 超长会话（30万+ token） | 开新会话；或调低 `auto_compact_token_limit` |
| 403 (Cloudflare) | 请求 UA 被拦 | 用浏览器 UA（脚本已带）；勿用裸 urllib 默认 UA |

---

## 七、还原 / 回退到原生 Codex

**一键还原**（恢复到安装前状态，包括 config.toml / models.json / OCR 技能）：

```bash
bash ~/.codex/opencode-codex-setup/setup-codex-opencode.sh --restore
```

脚本根据安装时记录的 `manifest.txt` 智能判断：
- 安装前有 `config.toml` → 从备份恢复；没有 → 删除
- 安装前有 `models.json` → 从备份恢复；没有 → 删除
- 恢复 `layout-ocr` / `smart-ocr` 技能
- 删除 `describe-image` 技能和 `describe_image.py`

**手动还原**（备份在 `~/.codex/backup-opencode-codex/`）：
```bash
cp ~/.codex/backup-opencode-codex/config.toml ~/.codex/config.toml
cp ~/.codex/backup-opencode-codex/models.json ~/.codex/models.json
mv ~/.codex/backup-opencode-codex/disabled-skills-ocr/layout-ocr ~/.agents/skills/
mv ~/.codex/backup-opencode-codex/disabled-skills-ocr/smart-ocr ~/.agents/skills/
rm -rf ~/.agents/skills/describe-image ~/.codex/skills/describe-image
```

> 还原后【重启 Codex】生效。如果只是想换主模型（不改架构），直接改 `config.toml` 的 `model` 字段即可。

---

## 八、安全提示

- **API Key**：写入 `config.toml` 的 `experimental_bearer_token`，权限建议 `chmod 600 ~/.codex/config.toml`
- **识别内容**：`describe_image.py` 把图片 base64 发送到 opencode.ai——敏感图片请知悉
- **脚本内嵌了你的 API Key 模板**：分享脚本前注意脱敏（`experimental_bearer_token` 是运行时输入，不会硬编码进脚本）
