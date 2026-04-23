# bunq Agent Integration Skill — Agent Reproduction Prompt

**Goal:** Create a Hermes skill that locally mirrors bunq's full developer documentation (~200 pages) with zero external parsers.

**Output:** `~/.hermes/skills/productivity/bunq-agent-integration/`

**Constraint:** Only `requests` allowed. No `bs4`, `lxml`, or `html.parser`.

---

## Steps

1. **Discover pages**
   - Fetch `https://doc.bunq.com/llms.txt`
   - Parse into `references/source-pages.csv` with columns: `url,slug,topic,status`
   - `slug` = URL path after `/`, replace `/` with `-`, lowercase
   - `topic` = first path segment
   - Expect ~200 rows

2. **Create fetch script** (`scripts/fetch_bunq_docs.py`)
   - Read CSV, loop each row
   - `GET` with Chrome UA, timeout 30s
   - Extract `<title>` and `<main>...</main>` (fallback to full HTML)
   - Strip `<script>`, `<style>`, all HTML tags via regex
   - Collapse 3+ newlines → 2, trim whitespace
   - Write to `references/pages/<slug>.md`

3. **Create index builder** (`scripts/build_all_pages_index.py`)
   - Read CSV, group by `topic`
   - Write `references/all-pages.md` with topic sections

4. **Create SKILL.md**
   - Frontmatter with `name: bunq-agent-integration`, `category: productivity`
   - Routing rules: load order = SKILL.md → all-pages.md → topic note → raw snapshot
   - Verified facts:
     - API Key = full access, no read-only mode
     - Standard `POST /payment` can execute without app confirmation
     - Only `draft-payment` guarantees app confirmation
     - Auth chain: `installation` → `device-server` (IP-bound) → `session-server`
     - Account-scoped paths work; user-level paths often 404

5. **Run & verify**
   ```bash
   python scripts/fetch_bunq_docs.py
   python scripts/build_all_pages_index.py
   ```
   - Verify `pages/` count matches CSV rows
   - Verify no zero-byte files

6. **Create topic reference notes**
   - `api-key-flow.md` — installation → device → session sequence
   - `oauth-flow.md` — grant flow, scopes, redirect URI
   - `production-and-security.md` — IP whitelist, credential storage (outside skill dir)

Done. No secrets in the skill directory. Keep production creds in `~/.bunq-prod/` or similar private path.
