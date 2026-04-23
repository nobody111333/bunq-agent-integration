# bunq Agent 集成 — 复现提示词

[→ English README](./README.md)

**目标：** 在本地完整镜像 bunq 开发者文档，然后指导人类完成 agent 与 bunq 账户的绑定。

**兼容 Agent：** Hermes、OpenClaw、Codex（OpenAI）、Claude Code、OpenCode、Kimi Code，或任何从本地目录加载技能的 agent 框架。

**输出目录：** `~/<agent-config-dir>/skills/productivity/bunq-agent-integration/`

- **Hermes** / **OpenClaw**：`~/.hermes/skills/productivity/bunq-agent-integration/`
- **Claude Code**：`.claude/skills/bunq-agent-integration/`（或项目级 `CLAUDE.md`）
- **Codex / OpenCode / Kimi Code**：遵循各 agent 的 skill/plugin 目录规范

**约束：** 只允许使用 `requests`。禁止安装 `beautifulsoup4`、`lxml`，禁止使用 `html.parser`。

> 重要说明：官方文档与官方发布的 OpenAPI / 机器可读规范才是首要事实来源。本仓库中的提示词与说明仅作为 agent 的执行层与路由层，不应被视为权威来源。若提示词与官方文档冲突，以官方文档为准。

## 另见 — 欧洲银行 API 索引

- [个人开发者友好的银行 API](./banks/personal-friendly.md) — bunq / Monzo / Starling / Revolut
- [不支持个人直接调用的银行 API](./banks/unsupported-personal.md) — ING / Deutsche Bank / BBVA / Santander / N26 / ABN AMRO / Nordea / Danske Bank / La Banque Postale / Rabobank / BNP Paribas / Société Générale

---

# Part A — bunq（详细步骤，已按本地原始 skill 复核）

bunq 已完整实践验证，以下是可复现的详细步骤。

## 第一阶段 — 镜像文档

### 步骤 1：发现页面

抓取 `https://doc.bunq.com/llms.txt`（bunq 官方发布，比 `sitemap.xml` 更干净）。

解析为 `references/source-pages.csv`，列：
```csv
url,slug,topic,status
```

规则：
- `slug` = URL 中 `/` 后的路径，将 `/` 替换为 `-`，小写
- `topic` = 第一个路径段（如 `api-reference`、`concepts`、`oauth`）
- `status` = `llms-indexed`

预期：约 200 行。

### 步骤 2：创建抓取脚本

创建 `scripts/fetch_bunq_docs.py`，必须：
1. 读取 `references/source-pages.csv`
2. 每行用 Chrome UA `GET`，超时 30s
3. 提取 `<title>` 作为页面标题
4. 提取 `<main>...</main>` 内容（无 `<main>` 时回退到完整 HTML）
5. 用正则过滤 `<script>`、`<style>`、所有 HTML 标签
6. 3 个以上连续换行压缩为 2 个，去除尾部空白
7. 写入 `references/pages/<slug>.md`
8. 逐 URL 打印进度；错误输出到 stderr；失败继续
9. 最后打印 `done: wrote snapshots into references/pages/`

### 步骤 3：创建索引生成器

创建 `scripts/build_all_pages_index.py`：
1. 读取 `references/source-pages.csv`
2. 按 `topic` 分组
3. 写入 `references/all-pages.md`，按主题列出 `slug → url`

### 步骤 4：运行并验证

```bash
python scripts/fetch_bunq_docs.py
python scripts/build_all_pages_index.py
```

验证：
- `references/pages/` 文件数与 CSV 行数一致
- 无零字节文件

---

## 第二阶段 — 构建 Skill 骨架

### 步骤 5：创建技能路由文件

不同 agent 使用不同的入口文件命名：

| Agent | 入口文件 | Frontmatter 格式 |
|-------|---------|-----------------|
| Hermes / OpenClaw | `SKILL.md` | YAML frontmatter |
| Claude Code | `CLAUDE.md` | Markdown（无严格 frontmatter） |
| Codex / OpenCode / Kimi Code | `README.md` 或 agent 专属配置 | Tool description 或 system prompt |

Hermes / OpenClaw 使用以下 frontmatter：
```yaml
---
name: bunq-agent-integration
description: Build and maintain bunq-to-agent integrations. Keeps a local indexed mirror of bunq developer docs, with on-demand fetch/update scripts and topic-organized reference files.
version: 0.1.0
author: Agent
license: MIT
metadata:
  hermes:
    tags: [bunq, banking, api, oauth, agent, integration, docs]
    category: productivity
---
```

Claude Code 创建 `CLAUDE.md`，包含：
- 清晰描述此 skill 的作用
- 何时使用（bunq 集成、认证问题、支付流程）
- 文件布局参考
- 下方列出的关键事实

Codex / OpenCode / Kimi Code 将内容适配为各自 agent 的 tool description 或 system prompt 格式。关键是向 agent 暴露**路由规则**和**已验证事实**。

正文必须包含（无论哪种 agent）：
- **触发条件** — 何时加载此 skill
- **内容清单** — 文件/脚本列表及作用
- **运作规则** — 入口文件仅作路由，详情在 `references/`
- **Agent 优先调用模式** — 加载顺序：入口文件 → all-pages.md → topic note → raw snapshot
- **任务速查** — 常见问题对应到具体参考文件
- **关键实践事实**（经生产和文档验证）：
  - bunq API Key = **完全访问**，无只读模式
  - 标准 `POST /payment` 可在无 app 确认的情况下执行（已实测 €1 即时到账）
  - 只有 `draft-payment` 强制 app 确认
  - `daily_limit_without_confirmation_login` 是会话级软阈值（如 €250），不是每日上限
  - 认证链：`installation` → `device-server`（绑定 IP）→ `session-server`
  - 账户级路径有效；用户级路径如 `/user/{id}/payment` 常返回 404
  - 生产集成流程：获取公网 IPv4 → 安全存储 API key → 生成 RSA 密钥对 → `POST /installation` → `POST /device-server`（带 VPS IPv4）→ `POST /session-server`
- **推荐工作流** — agent 应如何使用此 skill

### 步骤 6：创建主题参考笔记

在 `references/` 下写入以下简明主题文件：

| 文件 | 内容 |
|------|------|
| `index.md` | 本地文档地图；说明如何刷新文档 |
| `overview.md` | 高层起点（API Key vs OAuth、沙盒 vs 生产） |
| `api-key-flow.md` | installation → device-server → session-server 流程；IP 绑定；RSA 密钥对生成 |
| `oauth-flow.md` | OAuth 授权流程；scope；redirect URI 要求 |
| `production-and-security.md` | 切生产；IP 白名单；凭据存储规则（放在 skill 目录外，如 `~/.bunq-prod/`） |
| `read-only-capabilities.md` | 已确认的可读资源及可用检索模式 |
| `live-integration-notes.md` | 生产验证的本地 bunq 集成事实 |

每份主题文件 < 100 行。不重复原始页面内容；仅总结导航并补充实践笔记。

---

## 第三阶段 — 指导人类绑定 bunq 账户

skill 骨架完成后，向人类展示以下精确清单。不要自己执行这些步骤 — 必须由人类完成，因为它们涉及手机 app 交互和 secret 处理。

注意：bunq app 里的动作只是**创建 API key**。真正的生产环境接入是由 agent / 工具链通过 `installation` → `device-server` → `session-server` 完成的。

### 起飞前检查

1. **确认 agent 宿主机的公网 IPv4**
   ```bash
   curl -s https://ipinfo.io/ip
   ```
   人类必须记录此 IP。bunq 将把 API key 绑定到该 IP。

2. **创建安全存储目录**
   ```bash
   mkdir -p ~/.bunq-prod && chmod 700 ~/.bunq-prod
   ```

### 人类在 bunq 手机 app 中的操作

3. **生成 API Key**
   - 打开 bunq app → 个人资料 → 安全 → API keys
   - 点击「创建 API Key」
   - 立即复制 key（仅显示一次）
   - 保存到 `~/.bunq-prod/api_key`，权限 `chmod 600`

4. **记录免确认日限额**
   - 个人资料 → 安全 → 限额
   - 记录 `daily_limit_without_confirmation_login`（如 €250.00）
   - 这是**会话级软阈值**，不是每日上限

### Agent 辅助引导（人类执行命令，agent 协助排错）

5. **生成 installation 用 RSA 密钥对**
   ```bash
   openssl genrsa -out ~/.bunq-prod/installation.key 2048
   openssl rsa -in ~/.bunq-prod/installation.key -pubout -out ~/.bunq-prod/installation.pub
   ```

6. **调用 `POST /installation`**
   - 使用步骤 5 的公钥
   - 保存返回的 `Token` 和 `ServerPublicKey`

7. **调用 `POST /device-server`**
   - 传入步骤 3 的 API key
   - 传入步骤 1 的 **agent 宿主机公网 IPv4**
   - 此步骤将 key 绑定到 VPS IP

8. **调用 `POST /session-server`**
   - 使用 installation token + 私钥签名
   - 安全保存 session token

9. **验证只读访问**
   - `GET /user/{user_id}` — 应返回用户信息
   - `GET /user/{user_id}/monetary-account` — 应列出账户
   - `GET /user/{user_id}/monetary-account/{account_id}/payment` — 应列出交易
   - 若返回 404，说明路径错误（大概率缺少账户级 scope）

### 必须口头朗读的安全警告

- **API Key = 完全访问**。服务端不存在只读模式。
- **标准 `POST /payment` 可在免 app 确认的情况下执行**，只要金额低于会话限额。
- **只有 `draft-payment` 强制 app 确认。** 若人类要求每笔付款都需手动审批，必须使用 `draft-payment`。
- **将 `~/.bunq-prod/` 排除在 git 外。** 若主目录已版本化，请加入 `.gitignore`。
- **若宿主机 IP 变更**，必须重新注册 device-server。

---

# Part B — Revolut（研究笔记）

> ⚠️ **状态：已研究，未亲自验证。** Revolut 不为标准个人账户提供直接 API key。以下内容根据官方文档和社区报告整理。

## API 全景概览

Revolut 运营**多个独立的 API 产品**，各自面向不同用户群体，访问要求也不同：

| API 产品 | 目标用户 | 个人账户可用？ | 认证方式 | 文档 |
|---------|---------|-------------|---------|------|
| **Business API** | Revolut Business 客户 | ❌ 需要 Business 账户 | OAuth 2.0 + 客户端证书 | https://developer.revolut.com/docs/business |
| **Merchant API** | 电商商户 | ❌ 需要 Merchant 账户 | API key (secret) | https://developer.revolut.com/docs/merchant |
| **Open Banking API** | 持牌 TPP（AISP/PISP） | ❌ 需要 PSD2 牌照 | OAuth 2.0 + eIDAS 证书 | https://developer.revolut.com/docs/open-banking |
| **Crypto Ramp API** | 加密货币服务集成商 | ❌ 需要合作资质 | API key | https://developer.revolut.com/docs/crypto-ramp |
| **Revolut X** | 交易/交易所合作伙伴 | ❌ 邀请制 | OAuth 2.0 | https://developer.revolut.com/docs/revolut-x |

**个人用户关键发现：** Revolut 标准个人账户**没有直接 API key**。与 bunq 不同，Revolut 不会在消费者手机 app 中提供「API Keys」入口。

## 个人账户的变通方案

### 方案 1：升级到 Revolut Business

- 在 https://business.revolut.com/ 注册
- 月费因套餐而异
- 获批后进入 Developer Portal 生成 API 凭据
- 完整 Business API 访问：账户、付款、卡片、团队管理、外汇

### 方案 2：Open Banking 聚合器（只读）

个人用户可通过持牌 Open Banking 聚合器**间接**访问 Revolut 数据：

| 聚合器 | 覆盖范围 | 费用 | 说明 |
|--------|---------|------|------|
| **Plaid** | 欧盟 + 英国 | 免费起步 | Revolut 支持良好，文档完善 |
| **TrueLayer** | 欧盟 + 英国 | 免费起步 | 英国原生，Revolut 集成快速 |
| **Tink** | 欧盟 | 免费起步 | 瑞典公司，北欧强势 |
| **GoCardless** | 欧盟 + 英国 | 免费起步 | 专注付款/Variable Recurring Payments |
| **Nordigen / GoCardless** | 欧盟 | 免费 tier | 账户信息服务（只读） |

**运作方式：**
1. 你在聚合器注册（不在 Revolut）
2. 聚合器是持牌 AISP，可通过 PSD2 访问 Revolut
3. 用户通过 OAuth 授权流程同意访问
4. 聚合器返回标准化的账户/交易数据
5. 你的 agent 调用聚合器 API，而非直接调用 Revolut

**限制：**
- 多数聚合器只读（AISP scope）
- 发起付款需要 PISP 牌照（门槛更高）
- 数据新鲜度不一（通常是批量，非实时）
- 多一层中间商和成本

### 方案 3：CSV 导出 + 解析

Revolut 手机 app 支持：
- 月度账单导出（PDF/CSV）
- 交易搜索和筛选

这完全不需要 API，但完全手动。agent 可以在人类提供 CSV 后进行解析。

## Business API 深入（如果你有 Business 账户）

### 认证流程

1. 在 Revolut Developer Portal **注册应用**
2. **生成客户端证书**（沙盒和生产分开）
3. **上传证书** 授权应用访问你的 Revolut Business 账户
4. 通过 OAuth 2.0 + client assertion（用你的证书签名的 JWT）**获取 access token**
5. 使用 Bearer token **调用 API**

### 核心端点

| 资源 | 端点模式 | 能力 |
|------|---------|------|
| 账户 | `GET /accounts` | 列出商业账户 |
| 收款人 | `GET /counterparties` | 管理付款对象 |
| 付款 | `POST /transfer` | 发起转账 |
| 付款草稿 | `POST /payment-drafts` | 创建待审批草稿 |
| 交易 | `GET /transactions` | 列出交易 |
| 卡片 | `GET /cards` | 管理团队卡片 |
| 外汇 | `POST /exchange` | 货币兑换 |
| 团队 | `GET /team-members` | 管理商业团队 |

### 沙盒

- 独立环境：`https://sandbox-b2b.revolut.com`
- 模拟充值、交易、付款流程
- 不涉及真实资金

## OpenAPI 规范

Revolut 在 GitHub 发布机器可读规范：
- **仓库：** https://github.com/revolut-engineering/revolut-openapi
- **格式：** JSON 和 YAML
- **文件：** `business.yaml`、`open-banking.yaml`、`merchant-*.yaml`、`crypto-ramp-*.yaml`、`revolut-x.yml`

可导入 Postman、Swagger，或与 OpenAPI Generator 配合生成 SDK。

## 对比：bunq vs Revolut 的 Agent 集成

| 维度 | bunq | Revolut |
|------|------|---------|
| **个人 API key** | ✅ App 内可用 | ❌ 不可用 |
| **使用 API 所需账户** | 任意账户 | Business 账户 |
| **认证复杂度** | API Key + RSA 密钥对 | OAuth 2.0 + 客户端证书 |
| **只读模式** | ❌ 无（完全访问） | ❌ 无（Business = 完全） |
| **IP 绑定** | ✅ 需要（device-server） | ❌ 不需要 |
| **沙盒** | ✅ 有（sandbox API + sandbox user/key 流程） | ✅ 有（Business） |
| **Open Banking** | ❌ 无 | ✅ 有（通过 TPP） |
| **文档质量** | 良好，有 `llms.txt` | 良好，有 OpenAPI 规范 |
| **Webhook** | ❌ 无 | ✅ 有 |
| **多币种** | 有限 | ✅ 原生（30+） |
| **实时同步** | 仅轮询 | Webhook 可用 |

## 个人用户建议

如果你有 **Revolut 标准个人账户**，想要 agent 集成：

1. **短期：** 使用 Open Banking 聚合器（Plaid/TrueLayer）获取只读访问
2. **中期：** 若自动化节省的价值超过月费，考虑升级到 Revolut Business
3. **替代方案：** 用 bunq 做直接 API 访问，Revolut 仅用于消费/卡片功能

若选择 **Revolut Business**，skill 构建模式与 bunq 类似：
1. 镜像文档（Revolut 提供 OpenAPI 规范，非 `llms.txt`）
2. 构建带路由规则的 skill 骨架
3. 指导人类完成 Business 账户注册 → Developer Portal → 证书生成 → OAuth 流程

---

## 完成标准

- [ ] `references/source-pages.csv` 约 200 行（bunq）
- [ ] `references/pages/` 中 `.md` 文件数与 CSV 行数一致（bunq）
- [ ] `fetch_bunq_docs.py` 仅依赖 `requests` 即可运行
- [ ] `all-pages.md` 已生成且按主题分组
- [ ] Skill 入口文件包含正确的路由规则 + 已验证事实
- [ ] `references/` 下存在主题参考笔记
- [ ] skill 目录内无任何生产 secret
- [ ] 人类已完成绑定清单或明确推迟
