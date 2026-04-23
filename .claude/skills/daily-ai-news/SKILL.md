---
name: daily-ai-news
description: Generate a daily Thai-language AI news brief from the last 24 hours of trusted sources, enrich it with three expert perspectives (professor / AI specialist / professional programmer), commit it to this repository via the GitHub connector, and push a TL;DR to LINE. Use this when the user asks for a "daily AI news brief", a "สรุปข่าว AI วันนี้", triggers this skill by name, or when it runs on schedule via Claude Web Routine.
---

# Daily AI News Brief

End-to-end routine that produces **one Markdown article per day** at `articles/YYYY-MM-DD-brief.md`, commits it via the GitHub connector, then notifies LINE. Runs entirely inside Claude (Web / Routine) — **no shell, no git CLI, no Bash**.

## Runtime contract

- **Timezone:** Asia/Bangkok. Compute `YYYY-MM-DD` from this TZ.
- **Tools allowed:** `WebSearch`, `WebFetch`, `Read`, `Write`, `Edit`, and the configured GitHub MCP connector.
- **Tools FORBIDDEN:** `Bash`, any shell, any `git` CLI. If you catch yourself reaching for `Bash`, stop — this skill must run on Claude Remote Routine where shell is unavailable.
- **No fabrication:** every news item MUST have a real, fetched URL from the trusted-source list in `reference/trusted-sources.md`. If you cannot verify a URL via `WebFetch`, drop the item.

## Required environment

**Deployment note — Cloud Environment `ClaudeBot_Line`**

In this project's Claude Web Routine deployment, all env vars below are provisioned through the Cloud Environment pack named **`ClaudeBot_Line`**. That pack must be selected on the Routine (bottom-right of the Routine editor — the cloud-icon dropdown) before the run starts. The skill itself doesn't care about the pack name — it only reads variable **names** — but if a run is misconfigured and attaches a different pack (e.g. a generic `Thannob` pack without LINE keys), the LINE vars will resolve to `***missing***` and Step 6 will be skipped. Always confirm the Routine shows `ClaudeBot_Line` as the active Cloud Environment.

| Var | Purpose | Required | Must come from |
|---|---|---|---|
| `GITHUB_OWNER` | GitHub account / org owning the target repo | yes | `ClaudeBot_Line` (or defaults.json fallback) |
| `GITHUB_REPO` | Target repo name | yes | `ClaudeBot_Line` (or defaults.json fallback) |
| `GITHUB_BRANCH` | Branch to commit on (default `main`) | no | `ClaudeBot_Line` (or defaults.json fallback) |
| `LINE_CHANNEL_ACCESS_TOKEN` | Messaging API channel token | no (skip LINE if absent) | **`ClaudeBot_Line` only — never defaults.json** |
| `LINE_TO` | Target userId / groupId / roomId | no (skip LINE if absent) | **`ClaudeBot_Line` only — never defaults.json** |

### Env resolution order (apply per variable, first hit wins)

1. **Cloud Environment injected into this runtime.** Scan the current context for the variable in any of these shapes:
   - A system / system-reminder message that declares env (e.g. a block containing `GITHUB_OWNER=...` or JSON with that key).
   - A template placeholder like `{{GITHUB_OWNER}}` in the invocation prompt that was **substituted** before the model read it — i.e. the prompt now shows a real value where the placeholder was.
   - An env-reading MCP tool if one is connected (e.g. a tool named like `read_env`, `get_secret`, `env_get`).
2. **Inline in the skill invocation prompt.** The caller may have written `LINE_CHANNEL_ACCESS_TOKEN = <real token>` directly in the run prompt. Use that.
3. **`reference/defaults.json`** — read with the `Read` tool. **Only** consult this for `GITHUB_OWNER`, `GITHUB_REPO`, `GITHUB_BRANCH`. Never read secrets from here.
4. **Missing.** For `GITHUB_*` this is a hard abort. For `LINE_*` it means `LINE_ENABLED=false`.

Do **not** invent values. Do **not** treat literal placeholder strings like `<from Routine env>`, `{{GITHUB_OWNER}}`, `$LINE_TO`, or `PASTE_REAL_TOKEN_HERE` as real values — those are signs substitution failed.

---

## Step 0 — Preflight (fail fast, log loudly)

1. **GitHub connector check.** Confirm the GitHub MCP connector is connected (look for tools whose names start with `mcp__github` / `mcp__Github` / similar — e.g. `create_or_update_file`, `push_files`, `get_file_contents`). If **none** are available:
   - Print: `ABORT: GitHub connector is not connected. Enable the GitHub MCP connector in Claude settings and re-run.`
   - **Stop immediately.** Do not research, do not write any files.
2. **Resolve env.** For each variable in the table above, walk the resolution order. `Read` `reference/defaults.json` exactly once and cache it for tier 3.
3. **Print a resolution table** so failures are obvious:
   ```
   Env resolution (Cloud Environment: ClaudeBot_Line expected):
     GITHUB_OWNER              = thannob       (source: cloud-env / ClaudeBot_Line)
     GITHUB_REPO               = dailyainews   (source: cloud-env / ClaudeBot_Line)
     GITHUB_BRANCH             = main          (source: cloud-env / ClaudeBot_Line)
     LINE_CHANNEL_ACCESS_TOKEN = ***set***     (source: cloud-env / ClaudeBot_Line)
     LINE_TO                   = ***set***     (source: cloud-env / ClaudeBot_Line)
   ```
   For secrets, print `***set***` or `***missing***` — never the actual value. If a var fell through to `defaults.json`, say so explicitly — that tells the operator the Cloud Env pack didn't inject it.
4. **LINE gate.** If either LINE var is `***missing***` → `LINE_ENABLED=false`, print `LINE: skipped (env missing)`, continue — the commit step must still run. Otherwise `LINE_ENABLED=true`.
5. **GitHub gate.** If `GITHUB_OWNER` or `GITHUB_REPO` is `***missing***` → abort with a clear log.
6. **Date.** Compute `TODAY = YYYY-MM-DD` in `Asia/Bangkok`. Use this string for the filename, commit message, and article body.

---

## Step 1 — Research (last 24h, 5 stories)

Open `reference/trusted-sources.md` and treat it as the allow-list. Prefer sources from that list; do **not** invent outlets.

### 1a. Gather candidates with `WebSearch`

Use `WebSearch` with queries like:
- `AI news` with a date filter for the last 24h (include Thai queries: `ข่าว AI`, `ปัญญาประดิษฐ์ วันนี้`).
- Targeted queries per trusted source (e.g. `site:techcrunch.com AI`, `site:blognone.com AI`).

Capture for each candidate: URL, title, publisher (inferred from domain vs trusted-sources list), and the **search-result snippet** verbatim.

### 1b. Verification — tiered (handles egress-blocked runtimes)

Each story is assigned a verification tier. Do a quick `WebFetch` probe on one trusted URL (e.g. `https://example.com`) at the start of this step to detect whether WebFetch works at all in this runtime. Label the runtime:

- **`WEBFETCH_OK`** — probe returned 2xx.
- **`WEBFETCH_BLOCKED`** — probe returned 403 / network error (typical in Claude Web Routine today).

Then for each candidate:

| Tier | Requirements | Allowed when |
|---|---|---|
| **Tier 1 — Full fetch** | `WebFetch` returns 2xx for the URL; body contains the headline and a timestamp ≤ 24h old | `WEBFETCH_OK` |
| **Tier 2 — Search snippet** | URL's domain is on `reference/trusted-sources.md`; `WebSearch` result includes a substantive snippet and publishedTime (or a date in the snippet); **summary must paraphrase only what's in the snippet** | always — but only the sole option when `WEBFETCH_BLOCKED` |
| **Drop** | Can't satisfy Tier 1 or Tier 2 | — |

**Never** cite a URL that you could not at least see in a `WebSearch` result for a trusted-source domain. Search engines don't invent URLs, so a URL present in search results is real. The risk is **staleness of the snippet**, not fabrication — mitigate by never asserting more than the snippet says.

### 1c. Select exactly 5

Select **exactly 5** stories. Mix: aim for ≥1 Thai-language source and ≥3 international sources. Prefer Tier 1 over Tier 2 when both are available for the same candidate. Prefer primary announcements over commentary.

### 1d. Write `reference/sources.md`

Overwrite the file with this template — the `Verification` line is required:

```markdown
# Sources — {TODAY}

Generated: {TODAY} (Asia/Bangkok)
Runtime: {WEBFETCH_OK | WEBFETCH_BLOCKED}

1. **{Headline}**
   - Publisher: {Publisher}
   - URL: {final URL}
   - Published: {ISO timestamp or "per search snippet"}
   - Verification: {Tier 1 — WebFetch | Tier 2 — WebSearch snippet}
   - Summary: {1–2 sentences, strictly from fetched body or snippet}

2. ...
```

If you cannot find 5 items at Tier 1 or Tier 2, write however many you found and add:
```
> Note: only N items verified this run ({WEBFETCH_OK|WEBFETCH_BLOCKED}).
```
Do not pad with fiction.

If **zero** items were verified at any tier → skip Step 2–5, write a one-line stub to `articles/{TODAY}-brief.md` explaining the blocker, commit it (so the outage is visible), and still run Step 6 if `LINE_ENABLED`.

---

## Step 2 — Draft the article

Create the first draft in memory (don't commit yet). Structure:

```markdown
# สรุปข่าว AI ประจำวันที่ {TODAY}

> TL;DR
> - {bullet 1 — one sentence}
> - {bullet 2}
> - {bullet 3}

## ข่าวเด่น 24 ชั่วโมงที่ผ่านมา

### 1. {Headline}
{2–4 sentences. Link inline: [{Publisher}]({URL}). Every factual claim must trace to that URL.}

### 2. ...
```

Rules:
- Every story references its source URL at least once via inline markdown link.
- Do **not** invent quotes, numbers, dates, or names. If the source didn't say it (Tier 1 body) or if the snippet didn't say it (Tier 2), don't write it.
- Thai-first prose; technical terms can stay in English.
- **If any story is Tier 2**, add a small italic tag at the end of that story's paragraph:
  `_(อ้างอิงจาก search snippet — WebFetch ไม่สามารถเข้าถึงหน้าต้นทางใน runtime นี้)_`
- **If the whole article is Tier 2** (i.e. `WEBFETCH_BLOCKED`), prepend a blockquote immediately under the H1:
  ```
  > ⚠ WebFetch is unavailable in this Routine runtime. All stories below are cited from live search snippets on trusted-source domains; full-text verification was not possible. See `reference/sources.md` for tier breakdown.
  ```

---

## Step 3 — Three perspectives

For each of the 5 stories, produce a short reaction from **three personas**. Write them to `reference/perspectives.md` (overwrite) using this layout:

```markdown
# Perspectives — {TODAY}

## 1. {Headline}

**อาจารย์ (มหาวิทยาลัย):** {1–2 sentences — pedagogical framing, what students/readers should understand}
**ผู้เชี่ยวชาญด้าน AI:** {1–2 sentences — technical substance, caveats, what's genuinely new}
**โปรแกรมเมอร์มืออาชีพ:** {1–2 sentences — practical impact on day-to-day engineering work, tooling, cost}

## 2. ...
```

Keep each persona's voice distinct. No filler. No platitudes.

---

## Step 4 — Rewrite (integrated)

Produce the **final article body** by weaving the three perspectives into each story rather than listing them as separate blocks. Append an **Action items** section at the end.

Target structure for the final file `articles/{TODAY}-brief.md`:

```markdown
# สรุปข่าว AI ประจำวันที่ {TODAY}

> TL;DR
> - {bullet 1}
> - {bullet 2}
> - {bullet 3}

## ข่าวเด่น 24 ชั่วโมงที่ผ่านมา

### 1. {Headline} — [{Publisher}]({URL})
{2–4 sentences of reporting, then a paragraph that integrates the professor / AI-expert / programmer angles naturally. No "persona:" labels in the body.}

### 2. ...

## Action items

- **สำหรับอาจารย์/นักเรียน:** {1 concrete action}
- **สำหรับผู้เชี่ยวชาญ AI:** {1 concrete action}
- **สำหรับโปรแกรมเมอร์:** {1 concrete action}

---
_Generated by the `daily-ai-news` skill on {TODAY} (Asia/Bangkok). Verification mode: {WEBFETCH_OK | WEBFETCH_BLOCKED → WebSearch-snippet}._
```

Save the final markdown as a string in memory — call it `ARTICLE_BODY` — for the commit step.

---

## Step 5 — Commit via GitHub connector

**Never use git CLI.** Use the connector tool. The exact tool name depends on the connector build; common shapes:

- `create_or_update_file` with params: `owner`, `repo`, `path`, `branch`, `message`, `content` (plain UTF-8 string; connector handles base64 if needed), optional `sha` for updates.
- `push_files` with a `files` array.

### 5a. Idempotency check

Before writing, call the connector's `get_file_contents` (or equivalent) for `articles/{TODAY}-brief.md` on `branch`. Three outcomes:

- **Not found.** Create it. Proceed.
- **Found, content byte-for-byte identical to `ARTICLE_BODY`.** Skip the commit. Log: `Run status: NO-OP (idempotent) — today's brief already committed at <existing SHA>`. Capture that SHA as `COMMIT_SHA` and **still proceed to Step 6** (LINE may have been skipped last time).
- **Found, content differs.** Update with the returned `sha` — this is a meaningful re-run (e.g. the earlier run was `WEBFETCH_BLOCKED` and this one has fresh data).

### 5b. Commit

1. Set:
   - `path = articles/{TODAY}-brief.md`
   - `branch = GITHUB_BRANCH or "main"`
   - `message = "brief: {TOPIC} {TODAY} [verify={webfetch|search}]"` where `{TOPIC}` is a ≤40-char summary of the day's dominant theme (e.g. `OpenAI ships new agent API`), and `verify=` reflects the runtime label from Step 1b.
   - `content = ARTICLE_BODY`
2. Call the connector's create-or-update tool.
3. If the response includes a commit SHA (typical field: `commit.sha` or `sha`), capture it as `COMMIT_SHA`. Otherwise attempt to read it back by calling the connector's `get_file_contents` / commit-list tool for the branch.
4. Form `PERMALINK = https://github.com/{GITHUB_OWNER}/{GITHUB_REPO}/blob/{COMMIT_SHA}/articles/{TODAY}-brief.md`. Pin to the SHA — do **not** use `/blob/main/...` (that link drifts).
5. If the commit call fails: print the full connector error, do not retry silently, do not proceed to LINE.

---

## Step 6 — LINE push (optional)

Skip this step entirely if `LINE_ENABLED=false`.

Build the message text:

```
📰 สรุปข่าว AI {TODAY}

TL;DR
• {bullet 1}
• {bullet 2}
• {bullet 3}

อ่านฉบับเต็ม: {PERMALINK}
```

Send via **`WebFetch` with method POST** (no curl, no shell):

- URL: `https://api.line.me/v2/bot/message/push`
- Headers:
  - `Authorization: Bearer {LINE_CHANNEL_ACCESS_TOKEN}`
  - `Content-Type: application/json`
- Body (JSON):
  ```json
  {
    "to": "{LINE_TO}",
    "messages": [{ "type": "text", "text": "<msg>" }]
  }
  ```

On response:
- **HTTP 200:** print `LINE: ok`.
- **Non-200:** print `LINE ERROR: status={code} body={responseBody}`. **Do not retry.** The commit already landed — a failed notification is surfaced, not swallowed.
- **Network/egress error** (e.g. `WEBFETCH_BLOCKED` also blocks `api.line.me`, or the runtime rejects POST): print `LINE EGRESS BLOCKED: <error>`. Do not retry.

If LINE failed for any reason, also append a visible marker to the end of the committed article body and **update the commit** (via Step 5b update flow) so readers of the article know the notification didn't go out:

```
> 🔕 LINE notification for this brief was not delivered ({reason}). See the Routine log for the raw response.
```

This requires a second commit call — that's fine; do not retry LINE itself.

If `WebFetch` in the current environment doesn't allow POST with custom headers at all, say so explicitly and stop rather than falling back to a GET — never silently degrade.

---

## Step 7 — Final report

Print a compact summary to the user / routine log:

```
✅ Committed: articles/{TODAY}-brief.md @ {COMMIT_SHA_short}
   {PERMALINK}
{LINE status line}
```

---

## Error-handling summary

| Condition | Action |
|---|---|
| GitHub connector missing | Abort before any work. Log clearly. |
| `WEBFETCH_BLOCKED` (whole runtime) | Switch to Tier 2 (WebSearch snippet) for every story; banner at top of article; commit with `[verify=search]`. |
| <5 verifiable stories | Proceed with what you have; note the shortfall in sources.md. |
| URL unreachable / paywalled (single story, `WEBFETCH_OK` runtime) | Drop that item; try Tier 2 for it; if both fail, skip it. |
| GitHub commit fails | Surface the error. Do not continue to LINE. |
| Today's article already committed, content identical | NO-OP commit; still attempt LINE (it may have been skipped last time). |
| Today's article already committed, content differs | Update via the returned SHA. |
| LINE env missing | Skip step 6. Commit is still the success criterion. |
| LINE API non-200 | Log status + body. Append `🔕 LINE not delivered` marker to article, update commit. No retry. |
| LINE egress blocked (runtime can't reach api.line.me) | Same as non-200 — log, mark, no retry. |

## Files this skill touches

- `reference/sources.md` — overwritten each run (working artifact)
- `reference/perspectives.md` — overwritten each run (working artifact)
- `articles/{TODAY}-brief.md` — the single committed output

`reference/trusted-sources.md` and `reference/defaults.json` are **read-only** for this skill.
