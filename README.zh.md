# bunq Agent 集成 — 复现提示词

**目标：** 在本地完整镜像 bunq 开发者文档，然后指导人类完成 agent 与 bunq 账户的绑定。

**输出目录：** `~/.hermes/skills/productivity/bunq-agent-integration/`

**约束：** 只允许使用 `requests`。禁止安装 `beautifulsoup4`、`lxml`，禁止使用 `html.parser`。

---

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
7. 写入 `references/pages/<slug>.md`：
   ```markdown
   # {title}

   - URL: {url}
   - Slug: {slug}
   - Topic: {topic}

   ---

   {cleaned_text}
   ```
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

### 步骤 5：创建 SKILL.md

使用以下 frontmatter：
```yaml
---
name: bunq-agent-integration
description: Build and maintain bunq-to-agent integrations. Keeps a local indexed mirror of bunq developer docs, with on-demand fetch/update scripts and topic-organized reference files.
version: 0.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [bunq, banking, api, oauth, agent, integration, docs]
    category: productivity
---
```

正文必须包含：
- **触发条件** — 何时加载此 skill
- **内容清单** — 文件/脚本列表及作用
- **运作规则** — SKILL.md 仅作路由，详情在 `references/`
- **Agent 优先调用模式** — 加载顺序：SKILL.md → all-pages.md → topic note → raw snapshot
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

### 起飞前检查

1. **确认 agent 的公网 IPv4**
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
   - 选择环境：**Production**
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
   - 传入步骤 1 的 **agent 公网 IPv4**
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
- **若 VPS IP 变更**，必须重新注册 device-server。

---

## 完成标准

- [ ] `references/source-pages.csv` 约 200 行
- [ ] `references/pages/` 中 `.md` 文件数与 CSV 行数一致
- [ ] `fetch_bunq_docs.py` 仅依赖 `requests` 即可运行
- [ ] `all-pages.md` 已生成且按主题分组
- [ ] `SKILL.md` frontmatter、路由规则、实践事实齐全
- [ ] `references/` 下存在主题参考笔记
- [ ] skill 目录内无任何生产 secret
- [ ] 人类已完成绑定清单或明确推迟
