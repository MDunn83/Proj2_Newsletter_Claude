# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## MANDATORY: Read Before Building

Read `n8n_SKILL.md` completely before writing any node JSON. It encodes hard-won runtime lessons -- violating its rules produces workflows that import but fail silently.

Read `lessons_learned.md` before building. It captures project-specific issues and fixes discovered during prior builds.

Do not build anything until you confirm you have read both files.

---

## Constraints

- **GitHub MCP tools only.** All file pushes go through `mcp__github__push_files`. Do not run `git` commands locally or modify files on the local machine.
- **Target branch:** `REPLACE_WITH_BRANCH_NAME` on `REPLACE_WITH_REPO_NAME`
- The workflow JSON must be valid and importable into n8n without modification.
- After completing each build phase, push to GitHub and stop. Wait for explicit confirmation before proceeding to the next phase.
- If unsure how to implement any node or connection, stop and ask rather than guessing.

---

## Platform and Output

- **Automation platform:** n8n
- **Output:** Single JSON file importable into n8n (`REPLACE_WITH_WORKFLOW_FILENAME.json`)
- **LLM provider:** Groq (free tier)
- **Model:** `llama-3.3-70b-versatile`
- **Rate limiting:** Add a 3-second Wait node between LLM calls if processing multiple rows in rapid succession on Groq free tier

---

## n8n Credentials

| Service | Credential Name in n8n |
|---|---|
| Gmail | `Gmail OAuth2 API` |
| Google Sheets | `Google Sheets OAuth2 API` |
| Groq | `Groq account` |
| [Add others as needed] | |

---

## Global LLM Prompt Rule

Every LLM prompt must include this instruction verbatim:

```
Output ONLY the requested content. Begin directly with the first line of output.
Do not include any introductory text, preamble, or closing remarks.
```

---

## Google Sheets Structure

### [Primary Sheet Name] (trigger + update source)
| Column | Notes |
|---|---|
| REPLACE_WITH_COLUMN | |
| REPLACE_WITH_COLUMN | |
| Email | Recipient address for Gmail node -- always pull dynamically, never hardcode |

### [Log Sheet Name] (separate sheet, same Google Spreadsheet)
| Column | Notes |
|---|---|
| Timestamp | Use `$now.toISO()` |
| REPLACE_WITH_COLUMN | |
| Status | |

---

## Workflow Architecture

### Trigger
- **Testing:** Manual Trigger + Google Sheets `getRows` node (reads all rows at once; n8n iterates each row automatically via `runOnceForEachItem` Code nodes)
- **Production:** Swap Manual Trigger for REPLACE_WITH_PRODUCTION_TRIGGER -- no other nodes change

### Node Sequence

1. Describe your pipeline phases here, one node or group per line
2. Use this section as the single source of truth for Claude Code -- vague descriptions produce broken wiring
3. Name exact nodes and exact connections (e.g., "Route By Score True branch connects to BOTH Email Writer Chain AND Log node -- Gmail is a dead end")

### Batch vs. Single-Record Behavior

- **If the workflow must process ALL rows in a single run:** State that explicitly here. The dedup or suppression check is what prevents reprocessing -- not trigger filtering.
- **If the workflow processes one record at a time:** State that explicitly here.

---

## Key Architectural Decisions

Document your core design choices here. This section is where CLAUDE.md earns its value. Examples of what belongs here:

- **Suppression-before-LLM:** All eligibility and suppression checks belong upstream of every LLM call. If a check could result in not sending a message, it belongs before any LLM chain.
- **LLM context preservation:** After any `chainLlm` node, `$json` is dead. Use `$('NodeName').item.json` cross-node references in all downstream Code nodes.
- **No Loop Over Items:** n8n iterates rows natively via `runOnceForEachItem`. Loop Over Items adds confusion and is not needed.
- **Dedup pattern:** Use a Code node with a JavaScript Set for any "does this value exist in the log" check. Compare Datasets is a JOIN node, not a lookup node.
- **Gmail sequencing:** Gmail fires first (to confirm delivery), then Log, then any Update nodes. Gmail output carries no useful data -- all downstream nodes use cross-node references to upstream nodes.
- **Date comparisons:** Perform all date arithmetic in Code nodes using `Date.now()` and `new Date(value).getTime()`. Do not rely on n8n expression date helpers for threshold logic.
- **Sanitize Text node:** Insert a Code node between any LLM Chain and Gmail. Collapse single newlines into spaces while preserving paragraph breaks.
- **Sheet IDs:** Use placeholder `YOUR_GOOGLE_SHEET_ID` in exported JSON. Fill in manually after import.
- [Add project-specific decisions here]

---

## Phased Build Plan

Break the build into explicit phases. Claude Code commits after each phase and stops. Do not proceed to the next phase without explicit confirmation.

| Phase | Scope |
|---|---|
| Phase 1 | Trigger, input sheet read, initial data prep |
| Phase 2 | Core logic nodes (routing, filtering, dedup) |
| Phase 3 | LLM chains and sanitize nodes |
| Phase 4 | Output nodes (Gmail, Sheets append, update) |
| Phase 5 | End-to-end wiring review and placeholder audit |

---

## Post-Build Checklist

Before the session closes, confirm:

- [ ] All Sheet IDs replaced with `YOUR_GOOGLE_SHEET_ID` placeholder
- [ ] No hardcoded email addresses anywhere in the JSON
- [ ] No credential IDs or instance IDs in the JSON
- [ ] All cross-node references use exact node names as they appear in the JSON
- [ ] Mandatory read instructions still present in CLAUDE.md (Claude Code may overwrite this file)
- [ ] Final JSON pushed to target branch and confirmed importable
