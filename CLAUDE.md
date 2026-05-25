# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## MANDATORY: Read Before Building

Read `n8n_SKILL(2).md` completely before writing or editing any node JSON. It encodes runtime lessons (typeVersions, Code-node modes, LLM prompt syntax, IF-node fragility, dedup patterns) that prevent workflows that import but fail silently.

`CLAUDE_Starter.md` is a **generic, reusable template** for n8n builds — do not specialize it. This file is the concrete, project-specific guide.

(There is no `lessons_learned.md` in this repo, despite the starter referencing one.)

---

## What This Project Is

A single n8n workflow (`proj2_newsletter.json`) that produces a **synthesized daily AI-industry newsletter**: it reads 10 companies from a Google Sheet, pulls recent news per company, filters and classifies it, logs every decision, and emails a ≤5-paragraph briefing. Runs every 24h.

- **Platform:** n8n · **Output:** one importable workflow JSON (`proj2_newsletter.json`)
- **LLM:** Groq, model `llama-3.3-70b-versatile` (free tier)
- **News source:** Google News RSS (free, no key) via HTTP Request
- **Branch:** develop on `claude/wizardly-lamport-q3VKu` in `MDunn83/Proj2_Newsletter_Claude`
- **Pushing:** GitHub MCP only (`mcp__github__push_files`). Do not commit via local `git`.

---

## n8n Credentials

| Service | Credential name in n8n |
|---|---|
| Gmail | `Gmail OAuth2 API` |
| Google Sheets | `Google Sheets OAuth2 API` |
| Groq | `Groq account` |

---

## Global LLM Prompt Rule

Every LLM prompt ends with this verbatim:

```
Output ONLY the requested content. Begin directly with the first line of output.
Do not include any introductory text, preamble, or closing remarks.
```

---

## Google Sheet Structure (`Proj2_Claude.xlsx` → live Google Sheet)

### `Targets` (input, 10 rows)
| Column | Notes |
|---|---|
| Company Name | e.g. OpenAI, Anthropic, Cursor |
| Website URL | informational |
| Anchor | disambiguated Google News search term (e.g. "Cursor AI coding", "Elasticsearch AI") — used to build the RSS query AND as a relevance alias |
| Sector | topic-of-interest, passed to the classifier |

### `Log` (output + cross-run dedup source)
| Column | Notes |
|---|---|
| Company Name | |
| Signal Title | |
| Signal URL | **dedup key** — checked against incoming articles to prevent re-sending |
| Signal Type | one of the 8 categories |
| Summary | LLM 1-2 sentence summary |
| PubDate | from RSS |
| Logged | `={{ $now.toISO() }}` |
| Briefing Included | `Yes` (passed filters) / `No` (excluded) |
| Funding | LLM-extracted amount or `N/A` |

---

## Workflow Architecture

Triggers: **Manual Trigger** (testing) + **Schedule Trigger 24h** (production, 08:00 daily). Both feed `Config`.

```
Triggers → Config (recipientEmail) → Get Log → Get Targets
  → Build RSS URL (Anchor + "when:2d") → Fetch News (throttled HTTP)
  → Parse Articles (≤6/company, 48h window; Always Output Data)
  → Relevance Pre-filter → Filter & Dedup → Wait 3s
  → Classify (Groq) → Parse Classification
  → ├ IF Real ─true→ IF Include ─true→ Log Included / ─false→ Log Excluded
    └ Aggregate Included → IF Has Signals
          ├ true → Synthesize → Sanitize Text → Gmail Digest
          └ false → Gmail No News
```

Groq Chat Model connects via `ai_languageModel` to **both** `Classify` and `Synthesize`.

---

## Key Architectural Decisions

- **Always one email per run.** n8n skips a node when its input has 0 items, so an empty funnel (no articles / all duplicates / all irrelevant) would otherwise send nothing. `Parse Articles` has `alwaysOutputData: true`, and `Filter & Dedup` emits a **sentinel** item when nothing survives — keeping the Classify→Aggregate→email path alive. The sentinel is routed away from logging by `IF Real`. (Only exception: 0 rows in `Targets` → nothing runs.)
- **$100M rule = Funding only (Option A).** The threshold gates only `Funding`-category articles (`fundingMillions >= 100`). Partnerships and all other non-`Other` categories pass regardless of dollar amount. `Other` is always excluded.
- **Dedup is cross-run only**, keyed on `Signal URL` via a JS `Set` from `Get Log`. Intra-run duplicates are intentionally NOT removed — the same article can be a legitimate signal for two companies, and the synthesizer consolidates it anyway.
- **RSS rate-limit mitigation.** Fetch News uses `batchInterval: 1500` (one request at a time), a real `User-Agent`, `retryOnFail` (3×), and `onError: continueRegularOutput`. Bursting 10 bare requests is what triggers Google's 403s.
- **Recency via the query** (`when:2d`) plus a 48h safety filter in `Parse Articles` — makes it a true daily letter and shrinks volume before the LLM.
- **Company re-attachment after HTTP** is index-based in `Parse Articles` (10 responses in order). `continueOnFail` on Fetch News preserves alignment.
- **`chainLlm` kills `$json` downstream.** `Parse Classification` reads article fields via `$('Filter & Dedup').item.json`, not `$json`.
- **Groq rate safety** = `retryOnFail` + 5s backoff on both LLM nodes (the real throttle); `Wait 3s` adds spacing.
- **Sanitize preserves paragraphs.** Single newlines → spaces, double newlines → paragraph breaks; also emits an `html` field (`<br><br>`) for the Gmail body. Uses `String.fromCharCode` instead of escaped regex to avoid JSON double-escaping.
- **Recipient is never hardcoded.** Both Gmail nodes read `={{ $('Config').first().json.recipientEmail }}`; the placeholder lives only in the `Config` node.
- **Google Sheets nodes use `"resource": "sheetWithinDocument"`** with `operation` `getRows`/`append`, and `documentId`/`sheetName` as `"mode": "list"` resourceLocators. This n8n version requires the `resource` field and renders fine with it — do NOT strip it (the skill file's "never add resource" rule is outdated for this instance, verified against the live node).

---

## Post-Import Checklist

1. Fill `YOUR_GOOGLE_SHEET_ID` + the `Targets`/`Log` tab GIDs in all 4 Google Sheets nodes.
2. Set `recipientEmail` in the **Config** node.
3. Map credentials: `Google Sheets OAuth2 API`, `Groq account`, `Gmail OAuth2 API`.
4. Verify every IF node (`IF Real`, `IF Include`, `IF Has Signals`) — left side is an expression and the operator reads **"is true"** (most import-fragile part).
5. Confirm `Groq Chat Model` links to both `Classify` and `Synthesize`.
6. First run via Manual Trigger — confirm Google News RSS returns data (open network required; not testable in the GitHub-only cloud sandbox).
