# Hermes Agent Vision 组件配置指南

> 本文档针对 **Kimi Coding Plan** 等被 Hermes 默认黑名单限制的提供商，说明如何修复视觉识别功能。
>
> 适用版本：Hermes Agent >= 0.10.0
> 测试环境：Linux (Ubuntu), Hermes + kimi-k2.6 @ kimi-coding

---

## 目录

1. [问题背景](#1-问题背景)
2. [根因分析](#2-根因分析)
3. [解决方案](#3-解决方案)
   - 3.1 [快速修复：清空黑名单（推荐）](#31-快速修复清空黑名单推荐)
   - 3.2 [配置辅助视觉提供商（备选）](#32-配置辅助视觉提供商备选)
4. [验证方法](#4-验证方法)
5. [配置文件参考](#5-配置文件参考)
6. [故障排查](#6-故障排查)

---

## 1. 问题背景

在使用 Hermes Agent 进行图片分析时，如果主模型提供商是 `kimi-coding`（Kimi Coding Plan 端点），可能会遇到以下错误：

```
No LLM provider configured for task=vision provider=auto. Run: hermes setup
```

或模型回复：

```
I can't analyze the image right now.
```

**但实际上**，kimi-k2.6 模型本身具备视觉能力，且 Kimi Coding Plan 端点（`api.kimi.com/coding`）**已经支持图片输入**。问题出在 Hermes 代码中一个过于保守的黑名单。

---

## 2. 根因分析

Hermes 在 `agent/auxiliary_client.py` 中维护了一个不支持视觉的提供商黑名单：

```python
# agent/auxiliary_client.py (约第 298 行)
_PROVIDERS_WITHOUT_VISION: frozenset = frozenset({
    "kimi-coding",
    "kimi-coding-cn",
})
```

当主提供商在此黑名单中时，Hermes 会跳过该提供商的视觉调用，转而寻找辅助视觉提供商。如果没有配置辅助提供商，就会报错。

**关键点：**
- Kimi Coding Plan 使用 Anthropic Messages API 格式
- 早期该端点确实不支持 `image_url` 类型的消息内容
- 但当前版本（2025 年中之后）已支持图片输入
- 黑名单未同步更新，导致误杀

---

## 3. 解决方案

### 3.1 快速修复：清空黑名单（推荐）

直接修改本地 Hermes 源码，移除黑名单中的限制。

**步骤：**

```bash
# 1. 定位文件
HERMES_AGENT_DIR="$HOME/.hermes/hermes-agent"
FILE="$HERMES_AGENT_DIR/agent/auxiliary_client.py"

# 2. 备份原文件
cp "$FILE" "$FILE.bak.$(date +%Y%m%d)"

# 3. 清空黑名单（将 frozenset 内容置空）
sed -i 's/_PROVIDERS_WITHOUT_VISION: frozenset = frozenset({/_PROVIDERS_WITHOUT_VISION: frozenset = frozenset({/' "$FILE"
sed -i '/"kimi-coding",/d' "$FILE"
sed -i '/"kimi-coding-cn",/d' "$FILE"

# 4. 验证修改结果
grep -A2 "_PROVIDERS_WITHOUT_VISION" "$FILE"
```

**预期输出：**
```python
_PROVIDERS_WITHOUT_VISION: frozenset = frozenset({
})
```

**步骤 5：重启 Gateway（如使用 Web UI / 消息平台）**

```bash
hermes gateway restart
```

或在 CLI 中开始新会话：
```bash
hermes
# 然后输入 /reset
```

---

### 3.2 配置辅助视觉提供商（备选）

如果你不想修改源码，或主提供商确实不支持视觉，可以配置一个独立的辅助视觉提供商。

#### 选项 A：OpenRouter（推荐，模型选择丰富）

```bash
# 1. 获取 API Key: https://openrouter.ai/keys

# 2. 写入环境变量
echo "OPENROUTER_API_KEY=sk-or-v1-xxxxxxxx" >> ~/.hermes/.env

# 3. 配置辅助视觉
hermes config set auxiliary.vision.provider openrouter
hermes config set auxiliary.vision.model anthropic/claude-sonnet-4

# 4. 重启
hermes gateway restart
```

#### 选项 B：Google Gemini（免费额度充足）

```bash
# 1. 获取 API Key: https://aistudio.google.com/app/apikey

# 2. 写入环境变量
echo "GOOGLE_API_KEY=xxxxxxxx" >> ~/.hermes/.env

# 3. 配置辅助视觉
hermes config set auxiliary.vision.provider gemini
hermes config set auxiliary.vision.model gemini-2.5-flash

# 4. 重启
hermes gateway restart
```

#### 选项 C：Anthropic Claude

```bash
# 1. 获取 API Key: https://console.anthropic.com/

echo "ANTHROPIC_API_KEY=sk-ant-xxxxxxxx" >> ~/.hermes/.env

hermes config set auxiliary.vision.provider anthropic
hermes config set auxiliary.vision.model claude-sonnet-4

hermes gateway restart
```

---

## 4. 验证方法

### 4.1 检查黑名单状态

```bash
grep "_PROVIDERS_WITHOUT_VISION" ~/.hermes/hermes-agent/agent/auxiliary_client.py
```

### 4.2 检查辅助视觉提供商解析

```bash
cd ~/.hermes/hermes-agent
python3 -c "
from agent.auxiliary_client import resolve_vision_provider_client
provider, client, model = resolve_vision_provider_client()
print(f'Provider: {provider}')
print(f'Model: {model}')
print(f'Client ready: {client is not None}')
"
```

### 4.3 实际测试

在 Hermes 会话中发送一张图片，询问内容描述。如果正常返回图片内容分析，则配置成功。

---

## 5. 配置文件参考

### `~/.hermes/config.yaml` 相关配置

```yaml
model:
  default: kimi-k2.6
  provider: kimi-coding
  base_url: ''
  api_key: ''

auxiliary:
  vision:
    provider: auto      # auto 或指定 openrouter/gemini/anthropic
    model: ''           # 指定模型名称（provider=auto 时留空）
    base_url: ''        # 自定义端点（可选）
    api_key: ''         # 专用 API Key（可选，优先用 .env）
    timeout: 120        # 视觉分析超时（秒）
    extra_body: {}      # 额外请求参数
    download_timeout: 30  # 图片下载超时（秒）
```

### `~/.hermes/.env` 环境变量

```bash
# Kimi Coding Plan Key（主模型）
KIMI_API_KEY=sk-kimi-xxxxxxxx

# 辅助视觉提供商 Key（如使用辅助模式）
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxx
GOOGLE_API_KEY=xxxxxxxx
ANTHROPIC_API_KEY=sk-ant-xxxxxxxx
```

---

## 6. 故障排查

| 现象 | 可能原因 | 解决方案 |
|---|---|---|
| "No LLM provider configured for task=vision" | 黑名单限制 + 无辅助提供商 | 清空黑名单或配置辅助提供商 |
| "I can't analyze the image right now" | 提供商返回 400/404 | 检查黑名单；确认 API Key 有效 |
| 图片上传后无响应 | vision 工具未启用 | `hermes tools enable vision` |
| Web UI 图片分析失败但 CLI 正常 | Gateway 未重启 | `hermes gateway restart` |
| 辅助提供商配置后不生效 | 未重启或配置未保存 | 检查 `config.yaml`；确认 `/reset` 新会话 |

---

## 附：相关文件路径

```
~/.hermes/config.yaml              # 主配置
~/.hermes/.env                     # API Key 等敏感信息
~/.hermes/hermes-agent/agent/auxiliary_client.py   # 视觉黑名单源码
~/.hermes/logs/gateway.log         # Gateway 日志
```

---

*文档创建时间：2026-05-18*
*适用 Hermes 版本：>= 0.10.0*
