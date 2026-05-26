# Task 2 — n8n Workflow README
**Workflow:** Morning Brief – GitHub Trending → Discord Digest  
**File:** `Task2_Workflow_YourName.json`

---

## APIs Used & Why

| API | Endpoint | Reason |
|-----|----------|--------|
| **GitHub Search API** | `GET /search/repositories` | Free, no key required for public search. Returns rich structured repo data (stars, language, description) ideal for a trending digest. |
| **GitHub Contents API** | `GET /repos/{owner}/{repo}/readme` | Second enrichment call — fetches the README of the top-ranked repo to give context beyond just the repo name. |
| **Discord Webhook** | `POST {webhook_url}` | Zero-cost, instant delivery. Embed format renders beautifully for digest-style messages. |

---

## What the Transformation Does

1. **Filter & Sort** — The GitHub search already returns repos sorted by stars descending; the Code node re-sorts to guarantee ordering and slices to exactly the top 5 results.
2. **Field reshaping** — Only `name`, `description`, `stars`, `url`, `language`, and the README API URL are kept; all other GitHub fields (~40 keys) are dropped to keep the payload lean.
3. **README enrichment** — For each of the 5 repos, the raw README is fetched. Markdown syntax characters (`#`, `*`, `` ` ``, `>`, `[`, `]`) are stripped and the text is truncated to 300 characters to produce a readable plain-text snippet.
4. **Conditional branching** — An `IF` node routes repos into two Discord messages:
   - **Hot (≥ 1,000 ⭐)** → full embed with description + README snippet
   - **Rising (< 1,000 ⭐)** → compact bullet-list embed

---

## How the Error Path Behaves

Every HTTP node has `continueOnFail: true` with an **error output wired to a dedicated "Error Handler" node**.

- If the **GitHub search call** fails (rate-limit, timeout, DNS), the error branch sends a `🚨 Workflow Error` message to the same Discord channel so the team knows the brief was not generated.
- If a **README fetch** fails for a specific repo, the Code node catches the empty response and substitutes `"(README unavailable)"` — the digest still publishes for the other repos.
- If the **Discord POST** itself fails (webhook revoked, Discord outage), n8n logs the execution error in the execution history; the `continueOnFail` flag prevents the entire workflow from being silently dropped.

No secrets are hard-coded. The Discord Webhook URL is stored in an n8n **Credential** (`Discord Webhook` of type `HTTP Custom Auth`) and referenced via `$credentials.discordWebhookUrl`. The GitHub API is used unauthenticated (60 req/hour limit is sufficient for an hourly trigger).

---

## How to Import & Run

```bash
# 1. Start n8n
npx n8n  # or docker run -it --rm -p 5678:5678 n8nio/n8n

# 2. Import workflow
#    n8n UI → Workflows → Import from File → Task2_Workflow_YourName.json

# 3. Create credential
#    Settings → Credentials → New → HTTP Custom Auth
#    Name: "Discord Webhook"
#    Key: discordWebhookUrl  Value: https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN

# 4. Activate workflow or click "Test workflow" to run immediately
```

---

## Threshold Rationale

**1,000 stars** was chosen as the split threshold because:
- Repos above 1k stars have demonstrated community validation and are worth deeper coverage (full embed + README).
- Repos below 1k are "early signal" — worth surfacing but not worth the expanded format.
- The GitHub query (`created:>2024-01-01`) ensures recency, so even 200–900 star repos represent fast-rising new projects.
