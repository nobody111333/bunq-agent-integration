# bunq Agent Integration — Reproduction Prompt

[→ 中文说明](./README.zh.md)

**Goal:** Build a complete agent skill that mirrors bunq's developer documentation locally, then guide the human through binding their bunq account to the agent for live production use.

**Compatible agents:** Hermes, OpenClaw, Codex (OpenAI), Claude Code, OpenCode, Kimi Code, or any agent framework that loads skills from a local directory.

**Output directory:** `~/<agent-config-dir>/skills/productivity/bunq-agent-integration/`

- For **Hermes** / **OpenClaw**: `~/.hermes/skills/productivity/bunq-agent-integration/`
- For **Claude Code**: `.claude/skills/bunq-agent-integration/` (or project-local `CLAUDE.md`)
- For **Codex / OpenCode / Kimi Code**: follow your agent's skill/plugin directory convention

**Constraint:** Only `requests` is allowed. Do NOT install `beautifulsoup4`, `lxml`, or use `html.parser`.

## Also see — European Bank API Index

- [个人开发者友好的银行 API](./banks/personal-friendly.md) — bunq / Monzo / Starling / Revolut
- [不支持个人直接调用的银行 API](./banks/unsupported-personal.md) — ING / Deutsche Bank / BBVA / Santander / N26 / ABN AMRO / Nordea / Danske Bank / La Banque Postale / Rabobank / BNP Paribas / Société Générale

---

# Part A — bunq (Detailed, verified against local skill)

bunq 已完整实践验证，以下步骤已根据本地原始 skill 与实践笔记复核。

## Phase 1 — Mirror the Docs

### Step 1: Discover pages

Fetch `https://doc.bunq.com/llms.txt` (bunq publishes this; it is the cleanest source, better than `sitemap.xml`).

Parse it into `references/source-pages.csv` with columns:
```csv
url,slug,topic,status
```

Rules:
- `slug` = URL path after `/`, replace `/` with `-`, lowercase
- `topic` = first path segment (e.g. `api-reference`, `concepts`, `oauth`)
- `status` = `llms-indexed`

Expected: ~200 rows.

### Step 2: Create fetch script

Create `scripts/fetch_bunq_docs.py`. It must:
1. Read `references/source-pages.csv`
2. For each row, `GET` the URL with a standard Chrome UA, timeout 30s
3. Extract `<title>` as page title
4. Extract `<main>...</main>` body (fallback to full HTML if no `<main>`)
5. Strip `<script>`, `<style>`, all HTML tags via regex
6. Collapse 3+ newlines to 2, trim trailing whitespace
7. Write to `references/pages/<slug>.md`:
   ```markdown
   # {title}

   - URL: {url}
   - Slug: {slug}
   - Topic: {topic}

   ---

   {cleaned_text}
   ```
8. Print progress per URL; errors to stderr; continue on failure
9. End with `done: wrote snapshots into references/pages/`

### Step 3: Create index builder

Create `scripts/build_all_pages_index.py`:
1. Read `references/source-pages.csv`
2. Group rows by `topic`
3. Write `references/all-pages.md` with topic sections listing `slug → url`

### Step 4: Run & verify

```bash
python scripts/fetch_bunq_docs.py
python scripts/build_all_pages_index.py
```

Verify:
- `references/pages/` count matches CSV rows
- No zero-byte files

---

## Phase 2 — Build the Skill Skeleton

### Step 5: Create the skill routing file

Different agents use different naming conventions for the entrypoint:

| Agent | Entrypoint file | Frontmatter format |
|-------|----------------|-------------------|
| Hermes / OpenClaw | `SKILL.md` | YAML frontmatter |
| Claude Code | `CLAUDE.md` | Markdown (no strict frontmatter) |
| Codex / OpenCode / Kimi Code | `README.md` or agent-specific config | Tool description or system prompt |

For Hermes / OpenClaw, use this frontmatter:
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

For Claude Code, create `CLAUDE.md` with:
- A clear description of what this skill does
- When to use it (bunq integration, auth questions, payment flows)
- File layout reference
- Key facts listed below

For Codex / OpenCode / Kimi Code, adapt the content into your agent's tool description or system prompt format. The critical part is exposing the **routing rules** and **verified facts** to the agent.

Body must contain (regardless of agent):
- **Trigger conditions** — when to load this skill
- **What it contains** — list of files/scripts and their roles
- **Operating rule** — the entrypoint file is only routing; detailed content lives in `references/`
- **Agent-first call pattern** — load order: entrypoint → all-pages.md → topic note → raw snapshot
- **Fast path by task** — map common questions to specific reference files
- **Key learned facts** (verified against production and docs):
  - bunq API Key grants **full access**; there is **no read-only mode**
  - Standard `POST /payment` can execute without app confirmation (tested €1 instantly)
  - Only `draft-payment` guarantees app confirmation
  - `daily_limit_without_confirmation_login` is a per-session soft threshold (e.g. €250), not a daily cap
  - Auth chain: `installation` → `device-server` (IP-bound) → `session-server`
  - Account-scoped paths work; user-level paths like `/user/{id}/payment` often 404
  - For live integration: discover public IPv4 → store API key securely → generate RSA keypair → `POST /installation` → `POST /device-server` with VPS IPv4 → `POST /session-server`
- **Recommended workflow** — how an agent should use this skill

### Step 6: Create topic reference notes

Write these concise topic files under `references/`:

| File | Content |
|------|---------|
| `index.md` | Skill-local doc map; explain how to refresh docs |
| `overview.md` | High-level starting points (API Key vs OAuth, sandbox vs production) |
| `api-key-flow.md` | Installation → device-server → session-server sequence; IP binding; RSA keypair generation |
| `oauth-flow.md` | OAuth grant flow; scopes; redirect URI requirements |
| `production-and-security.md` | Moving to production; IP whitelist; credential storage rules (outside skill dir, e.g. `~/.bunq-prod/`) |
| `read-only-capabilities.md` | Confirmed readable resources and working retrieval patterns |
| `live-integration-notes.md` | Production-verified local bunq integration facts |

Each topic file < 100 lines. Do NOT duplicate raw page content; summarize navigation and add practical notes.

---

## Phase 3 — Guide Human to Bind bunq Account

After the skill skeleton is complete, present the human with this exact checklist. Do NOT perform these steps yourself — the human must do them because they involve mobile app interaction and secret handling.

Note: the bunq app step is just **creating the API key**. The actual production bootstrap is done by the agent/tooling via `installation` → `device-server` → `session-server`.

### Pre-flight checks

1. **Confirm the agent host's public IPv4**
   ```bash
   curl -s https://ipinfo.io/ip
   ```
   The human must record this IP. bunq will bind the API key to it.

2. **Create a secure storage directory**
   ```bash
   mkdir -p ~/.bunq-prod && chmod 700 ~/.bunq-prod
   ```

### Human actions in bunq mobile app

3. **Generate an API Key**
   - Open bunq app → Profile → Security → API keys
   - Tap "Create API Key"
   - Copy the key immediately (it is shown only once)
   - Save it to `~/.bunq-prod/api_key` with `chmod 600`

4. **Note the daily limit without confirmation**
   - Profile → Security → Limits
   - Record `daily_limit_without_confirmation_login` (e.g. €250.00)
   - This is a **per-session soft threshold**, not a daily cap

### Agent-assisted bootstrap (human runs commands, agent helps debug)

5. **Generate RSA keypair for installation**
   ```bash
   openssl genrsa -out ~/.bunq-prod/installation.key 2048
   openssl rsa -in ~/.bunq-prod/installation.key -pubout -out ~/.bunq-prod/installation.pub
   ```

6. **Call `POST /installation`**
   - Use the public key from step 5
   - Store returned `Token` and `ServerPublicKey`

7. **Call `POST /device-server`**
   - Pass the API key from step 3
   - Pass the **agent host's public IPv4** from step 1
   - This binds the key to the VPS IP

8. **Call `POST /session-server`**
   - Use installation token + private key to sign
   - Store session token securely

9. **Verify read-only access**
   - `GET /user/{user_id}` — should return user info
   - `GET /user/{user_id}/monetary-account` — should list accounts
   - `GET /user/{user_id}/monetary-account/{account_id}/payment` — should list transactions
   - If any return 404, the path is wrong (likely missing account scope)

### Safety warnings to read aloud

- **API Key = full access**. There is no server-side read-only mode.
- **Standard `POST /payment` can execute without app confirmation** for amounts under the session limit.
- **Only `draft-payment` guarantees app confirmation.** If the human wants manual approval for every payment, they must use `draft-payment`.
- **Keep `~/.bunq-prod/` out of git.** Add it to `.gitignore` if the home directory is versioned.
- **If the host IP changes**, the device-server must be re-registered.

---

# Part B — Revolut (Research Notes)

> ⚠️ **Status: researched, not personally verified.** Revolut does NOT offer a direct API key for standard personal accounts. The following is compiled from official docs and community reports.

## API Landscape Overview

Revolut operates **multiple distinct API products**, each with different target users and access requirements:

| API Product | Target User | Personal Account? | Auth Method | Portal |
|-------------|-------------|-------------------|-------------|--------|
| **Business API** | Revolut Business customers | ❌ Requires Business account | OAuth 2.0 + client certificate | https://developer.revolut.com/docs/business |
| **Merchant API** | E-commerce merchants | ❌ Requires Merchant account | API key (secret) | https://developer.revolut.com/docs/merchant |
| **Open Banking API** | Licensed TPPs (AISP/PISP) | ❌ Requires PSD2 license | OAuth 2.0 + eIDAS certificate | https://developer.revolut.com/docs/open-banking |
| **Crypto Ramp API** | Crypto service integrators | ❌ Requires partnership | API key | https://developer.revolut.com/docs/crypto-ramp |
| **Revolut X** | Trading/exchange partners | ❌ Invite-only | OAuth 2.0 | https://developer.revolut.com/docs/revolut-x |

**Key finding for personal users:** There is **no direct API key** for standard Revolut personal accounts. Unlike bunq, Revolut does not expose an "API Keys" section in the consumer mobile app.

## Personal-Account Workarounds

### Option 1: Upgrade to Revolut Business

- Sign up at https://business.revolut.com/
- Monthly fees apply (varies by plan)
- Once approved, access the Developer Portal to generate API credentials
- Full Business API access: accounts, payments, cards, team management, FX

### Option 2: Open Banking Aggregator (Read-only)

Personal users can access their Revolut data **indirectly** through regulated Open Banking aggregators:

| Aggregator | Coverage | Cost | Notes |
|------------|----------|------|-------|
| **Plaid** | EU + UK | Freemium | Strong Revolut support, good docs |
| **TrueLayer** | EU + UK | Freemium | UK-native, fast Revolut integration |
| **Tink** | EU | Freemium | Swedish, strong in Nordics |
| **GoCardless** | EU + UK | Freemium | Focused on payments/Variable Recurring Payments |
| **Nordigen / GoCardless** | EU | Free tier | Account Information Service (read-only) |

**How it works:**
1. You register with the aggregator (not with Revolut)
2. The aggregator is a licensed AISP, so it can access Revolut via PSD2
3. You redirect the user through OAuth consent flow
4. The aggregator returns normalized account/transaction data via its own API
5. Your agent calls the aggregator API, not Revolut directly

**Limitations:**
- Read-only for most aggregators (AISP scope)
- Payment initiation requires PISP license (higher bar)
- Data freshness varies (typically batched, not real-time)
- Additional dependency and cost layer

### Option 3: CSV Export + Parse

Revolut mobile app supports:
- Monthly statement export (PDF/CSV)
- Transaction search and filtering

This is zero-API, but completely manual. The agent can parse exported CSVs if the human provides them.

## Business API Deep Dive (if you have a Business account)

### Authentication Flow

1. **Register application** in Revolut Developer Portal
2. **Generate client certificate** (sandbox and production are separate)
3. **Upload certificate** to authorize your app
4. **Obtain access token** via OAuth 2.0 with client assertion (JWT signed by your certificate)
5. **Make API calls** with Bearer token

### Core Endpoints

| Resource | Endpoint Pattern | Capability |
|----------|-----------------|------------|
| Accounts | `GET /accounts` | List business accounts |
| Counterparties | `GET /counterparties` | Manage payment recipients |
| Payments | `POST /transfer` | Initiate transfers |
| Payment Drafts | `POST /payment-drafts` | Create drafts for approval |
| Transactions | `GET /transactions` | List transactions |
| Cards | `GET /cards` | Manage team cards |
| FX | `POST /exchange` | Currency exchange |
| Team | `GET /team-members` | Manage business team |

### Sandbox

- Separate environment at `https://sandbox-b2b.revolut.com`
- Simulate top-ups, transactions, payment flows
- No real money involved

## OpenAPI Specifications

Revolut publishes machine-readable specs on GitHub:
- **Repo:** https://github.com/revolut-engineering/revolut-openapi
- **Formats:** JSON and YAML
- **Files:** `business.yaml`, `open-banking.yaml`, `merchant-*.yaml`, `crypto-ramp-*.yaml`, `revolut-x.yml`

These can be imported into Postman, Swagger, or used with OpenAPI Generator for SDK creation.

## Comparison: bunq vs Revolut for Agent Integration

| Aspect | bunq | Revolut |
|--------|------|---------|
| **Personal API key** | ✅ Available in app | ❌ Not available |
| **Account needed for API** | Any account | Business account |
| **Auth complexity** | API Key + RSA keypair | OAuth 2.0 + client cert |
| **Read-only mode** | ❌ No (full access) | ❌ No (Business = full) |
| **IP binding** | ✅ Required (device-server) | ❌ No |
| **Sandbox** | ✅ Yes (sandbox API + sandbox user/key flow) | ✅ Yes (Business) |
| **Open Banking** | ❌ No | ✅ Yes (via TPP) |
| **Docs quality** | Good, has `llms.txt` | Good, has OpenAPI specs |
| **Webhook support** | ❌ No | ✅ Yes |
| **Multi-currency** | Limited | ✅ Native (30+) |
| **Real-time sync** | Pull only | Webhook available |

## Recommendation for Personal Users

If you have a **standard Revolut personal account** and want agent integration:

1. **Short term:** Use an Open Banking aggregator (Plaid/TrueLayer) for read-only access
2. **Medium term:** Consider upgrading to Revolut Business if the monthly fee is justified by automation savings
3. **Alternative:** Stick with bunq for direct API access, use Revolut for spending/card features only

If you proceed with **Revolut Business**, the skill-building pattern is similar to bunq:
1. Mirror the docs (Revolut has OpenAPI specs, not `llms.txt`)
2. Build the skill skeleton with routing rules
3. Guide the human through Business account setup → Developer Portal → certificate generation → OAuth flow

---

## Done Criteria

- [ ] `references/source-pages.csv` has ~200 rows (bunq)
- [ ] `references/pages/` has same count of `.md` files (bunq)
- [ ] `fetch_bunq_docs.py` works with only `requests` installed
- [ ] `all-pages.md` was generated and groups by topic
- [ ] Skill entrypoint file has correct routing rules + learned facts
- [ ] Topic reference notes exist under `references/`
- [ ] No production secrets anywhere in the skill directory
- [ ] Human has completed the binding checklist or explicitly deferred it
