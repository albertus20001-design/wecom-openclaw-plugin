---
name: wecom-doc-via-mcporter
description: 企业微信文档与智能表格操作，稳定走 mcporter 直连 `wecom-doc` MCP server，而不是依赖 `wecom_mcp` tool 注入。适用于用户提到企业微信文档、创建文档、写文档、编辑文档、智能表格、子表、字段、记录时，尤其适合当前运行环境里 `wecom_mcp` 不可见或不稳定的场景。
---

# 企业微信文档（通过 mcporter）

通过 `mcporter` 直接调用 `wecom-doc` MCP server，完成企业微信文档与智能表格操作。

## 何时使用

在以下场景使用本 skill：

- 用户要创建企业微信文档
- 用户要编辑企业微信文档正文
- 用户要创建或操作企业微信智能表格
- 当前环境中 `wecom_mcp` tool 不可见、不可用、或行为不稳定

默认将“创建文档 / 写文档 / 新建表格”理解为企业微信文档能力；无需再追问平台，除非用户明确指定别的平台。

## 执行原则

- 优先走 `mcporter`，不要依赖 `wecom_mcp`
- 会话中第一次操作前，先完成环境检查与 server 配置
- 如果中途需要用户授权或补配置，用户完成后要**自动续跑原始请求**，不要让用户重新描述
- 所有调用都使用 `mcporter ... --output json`

## 第一次操作前的检查

### 1. 检查 `mcporter` 是否已安装

执行：

```bash
which mcporter && mcporter --version
```

如果未安装，询问用户是否安装。用户同意后执行：

```bash
npm install -g mcporter
```

安装完成后继续后续步骤。

### 2. 读取企业微信文档 MCP 配置

执行：

```bash
cat ~/.openclaw/wecomConfig/config.json
```

要求存在：

- `mcpConfig.doc.type`
- `mcpConfig.doc.url`

如果缺失，告知用户当前企业微信文档 MCP 配置未生成，需要先让企业微信文档能力完成授权或初始化。

### 3. 配置 `wecom-doc` server

先检查：

```bash
mcporter list wecom-doc --output json
```

如果不存在，则从 `~/.openclaw/wecomConfig/config.json` 读取 `mcpConfig.doc.type` 和 `mcpConfig.doc.url`，执行：

```bash
mcporter config add wecom-doc --type "<type>" --url "<url>"
```

然后再次验证：

```bash
mcporter list wecom-doc --output json
```

## 通用调用方式

### 查看 `wecom-doc` server 可用工具

```bash
mcporter list wecom-doc --output json
```

### 调用某个工具

```bash
mcporter call wecom-doc.<tool> --args '<json>' --output json
```

## 核心任务

### 1. 创建文档

创建普通文档：

```bash
mcporter call wecom-doc.create_doc --args '{"doc_type":3,"doc_name":"文档名"}' --output json
```

创建智能表格：

```bash
mcporter call wecom-doc.create_doc --args '{"doc_type":10,"doc_name":"表格名"}' --output json
```

成功后保存返回：

- `docid`
- `url`

### 2. 编辑文档正文

使用 Markdown 全量覆写：

```bash
mcporter call wecom-doc.edit_doc_content --args '{"docid":"DOCID","content":"# 标题\n\n正文","content_type":1}' --output json
```

注意：

- `content` 直接传原始 Markdown 文本
- 这是**全量覆写**，不是追加

### 3. 智能表格工作流

典型顺序：

1. `create_doc(doc_type=10)` 获取 `docid`
2. `smartsheet_get_sheet` 获取 `sheet_id`
3. `smartsheet_get_fields` 查看默认字段
4. `smartsheet_update_fields` 重命名默认字段
5. `smartsheet_add_fields` 添加剩余字段
6. `smartsheet_add_records` 添加记录

## 返回结果处理

### 成功

若返回：

```json
{"errcode":0,"errmsg":"ok",...}
```

则继续使用结果字段回复用户；创建文档时优先返回：

- 文档标题
- 文档链接
- `docid`

### 授权错误

如果返回：

- `errcode = 850002`
- 且包含 `help_message`

则将 `help_message` **逐字原样**发给用户，不要改写，不要摘要。

用户完成授权后，直接重新执行原调用。

### 其他错误

如果 `errcode != 0`，优先原样展示：

- `errmsg`
- 以及有价值的错误字段

必要时重试 1 次；若仍失败，再告知用户失败信息。

## 推荐回复风格

- 创建成功：直接给链接 + docid
- 授权缺失：原样输出授权提示
- 配置缺失：直接说明是 `mcporter` / `wecom-doc` 配置缺失
- 不讲内部实现细节，除非用户在排障
