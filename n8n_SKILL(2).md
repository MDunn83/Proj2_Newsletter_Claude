# n8n_SKILL.md -- n8n Build Rules for Claude Code

## MANDATORY BEFORE BUILDING
Read this file completely before writing any node JSON.
Do not build anything until you confirm you have read it.

These rules encode hard-won lessons from production builds. Violating them produces workflows that import but fail at runtime.

---

## Trigger and Batch Processing

- Use a Manual Trigger + Google Sheets `getRows` node to read all rows at once when the workflow needs to process a list of items in a single run
- The operation name in the n8n UI is **Get Row(s)**, which maps to `"operation": "getRows"` in JSON. There is no `getAll` operation -- it does not exist and will cause the node to fail.
- Never add a `resource` field to a Google Sheets read node. Adding `"resource": "sheetWithinDocument"` (or any value) causes the node to render only the Resource selector in the UI -- all other parameters disappear and the node becomes unconfigurable.
- Never design for one-row-at-a-time trigger processing unless the use case genuinely requires it
- When deduplication is needed, read the log sheet separately and check in a Code node -- do not use the trigger to filter
- Google Sheets Trigger must use typeVersion 1 -- higher versions cause "Install this node" errors on import

---

## Node typeVersions

- Google Sheets Trigger: typeVersion 1
- Google Sheets (read/write): typeVersion 4
- Gmail: typeVersion 2
- HTTP Request: typeVersion 4.2
- Code: typeVersion 2
- IF: typeVersion 2
- LangChain chainLlm: typeVersion 1.4
- Always specify the n8n Cloud version in CLAUDE.md so typeVersions can be matched correctly

### Google Tasks -- Due Date Format
- Always set due dates in full ISO 8601 datetime format: `"2026-07-01T00:00:00.000Z"`
- The Google Tasks API rejects bare date strings like `"2026-07-01"` with a 400 error
- Never use `.split('T')[0]` on `toISOString()` calls when setting task due dates

---

## Code Node Rules

- Always specify mode explicitly: runOnceForAllItems or runOnceForEachItem
- In runOnceForEachItem mode:
  - Use $json to access the current item -- never $input.first().json
  - Return a single object: `return { json: {...} }` -- never `return [{ json: {...} }]`
  - Cross-node references like `$('NodeName').item.json` work but lose context after HTTP Request nodes -- avoid where possible
- In runOnceForAllItems mode:
  - Use $input.all() to access all items
  - Return an array: `return [{ json: {...} }, { json: {...} }]`
- Never use $input.first() -- it only ever returns the first item regardless of how many items are flowing
- Every Code node must open by explicitly assigning all needed fields off $json before referencing them:
  ```javascript
  let text_clean = $json.text_clean || "";
  let action_items = $json.action_items || [];
  ```
  Never reference a field name directly in a Code node without first pulling it off $json.

---

## LLM Prompt Syntax

- All LLM prompt expressions must be wrapped in {{ }} -- this applies even when the prompt field is set to expression mode
- Never use JavaScript backtick template literals with ${ } -- n8n does not evaluate them
- String concatenation with + is valid inside {{ }} expressions
- Correct format:
  ```
  {{ 'Static text ' + $json.company + ' more static text ' + $json.summary }}
  ```
- Incorrect formats that will fail silently:
  ```
  `Static text ${ $json.company } more text`
  'Static text ' + $json.company + ' more text'   (without {{ }} wrapper)
  ```
- For multi-line prompts with \n, keep everything inside a single {{ }} expression using string concatenation
- Always add explicit field labels and line breaks when passing multiple fields to a prompt.
  Every field gets its own labeled line -- labels are what make the data parseable by the model:
  ```
  First Name: {{ $json['First Name'] }}
  Last Name: {{ $json['Last Name'] }}
  Role: {{ $json.Role }}
  Start Date: {{ $json['Start Date'] }}
  ```

---

## LLM Prompt Behavior Rules

- Every structured or parseable LLM prompt must include this instruction verbatim:
  ```
  Output ONLY the requested content. Begin directly with the first line of output.
  Do not include any introductory text, preamble, or closing remarks.
  ```
  This applies to every LLM node in every workflow without exception. Add it to CLAUDE.md as a global instruction so it applies automatically.

- For list extraction, never use newline separation -- models frequently ignore it.
  Use `---` as the delimiter and instruct the model explicitly:
  ```
  Separate each item with this exact text on its own line: ---
  ```
  Split in the downstream Code node:
  ```javascript
  const items = text
    .split("---")
    .map(i => i.trim())
    .filter(i => i.length > 0);
  ```

- LLM output is always dirty. Always apply `.trim()` before parsing.
  Always implement a regex fallback to isolate the JSON block from any preamble.

- Some models emit literal `\n` strings (two characters) instead of actual newlines.
  Add this sanitization to any Code node that processes LLM output:
  ```javascript
  text = text.replace(/\\n/g, ' ');
  ```
  In a JSON workflow file this must be written as `\\\\n` to survive double-escaping.

- Models with a reasoning or thinking mode (e.g., qwen/qwen3-32b) output reasoning traces
  wrapped in `<think>...</think>` tags. Add a regex strip as a safety net in sanitization nodes:
  ```javascript
  text = text.replace(/<think>[\s\S]*?<\/think>/gi, '');
  ```
  Also add `"reasoning_effort": "none"` to every API request body for any model with a reasoning mode.

- When rendering an array in a Gmail body expression, always include an Array.isArray guard.
  Without it, undefined or malformed arrays render as `[object Object]` with no error signal.

- Funding amounts returned by LLMs use suffix notation ("$30B", "$500M"). Numeric comparison
  operators like >= will coerce these strings to NaN and always fail silently. Strip the suffix
  and multiply to a common unit before any comparison:
  ```javascript
  function parseFundingToMillions(str) {
    if (!str) return null;
    const match = str.replace(/,/g, '').match(/\$?([\d.]+)\s*([BMK]?)/i);
    if (!match) return null;
    const num = parseFloat(match[1]);
    const suffix = (match[2] || '').toUpperCase();
    if (suffix === 'B') return num * 1000;
    if (suffix === 'M') return num;
    if (suffix === 'K') return num / 1000;
    return num;
  }
  // Usage: parseFundingToMillions("$30B") === 30000, parseFundingToMillions("$500M") === 500
  ```

---

## LLM Output Sanitization for Email

Even with explicit formatting rules in the prompt, models frequently insert hard line breaks mid-sentence. Prompt instructions alone are not sufficient. Always add a dedicated Sanitize Text Code node immediately after every LLM Chain node that feeds an email output.

The sanitize node collapses single newlines into spaces while preserving intentional paragraph breaks:

```javascript
let text = $json.text || '';
text = text.replace(/\\n/g, ' ');        // literal backslash-n (two chars)
text = text.replace(/\n(?!\n)/g, ' ');  // single newlines mid-prose
text = text.replace(/ {2,}/g, ' ');     // collapse extra spaces
text = text.trim();
return { json: { text } };
```

In a JSON workflow file the first replace must be written as `\\\\n` to survive double-escaping.

- The sequence is always: **LLM Chain → Sanitize Text → Gmail → Log → Update**
- Never wire Gmail directly from an LLM Chain node
- Log nodes must pull the email body from the Sanitize Text node via cross-node ref, not from the LLM Chain node

---

## Data Flow Between Nodes

- HTTP Request nodes and file processing nodes strip upstream item context
- After any HTTP Request node, the item only contains the response data -- all upstream fields are gone
- chainLlm nodes also strip all upstream item context, including arrays passed in before the chain.
  If a chainLlm node is anywhere in the path, treat $json as dead downstream.
  Use `$('NodeName').item.json` cross-node references in every Code node after a chainLlm node.
- Always add a Code node or Edit Fields node after the last enrichment node to explicitly carry all fields forward onto the item
- Fields that must be carried forward: any ID, name, or contextual data needed by downstream nodes
- Downstream nodes must then use $json.fieldName -- not cross-node references -- to access this data
- Status sheet column mappings must pull from the Build Status Row node output ($json), not from
  the original trigger. Pulling from the trigger bypasses all enriched fields (check-in date,
  timestamp, status flags) so they are never written to the sheet.
- Example: after scraping a website and fetching news, add a Code node that packages content,
  news, company, website, contactName, role, and email onto a single item before passing to the LLM

### Execution Order Enforcement

- Cross-node references (`$('NodeName').all()`) only work if that node has already executed in the
  current run. If the referenced node has not yet executed, the reference returns empty data silently
  with no error.

- To force execution order without corrupting data flow: place the node that must run first in the
  main connection chain before the node that depends on it. Have the dependent node retrieve its
  actual working parameters via cross-node reference to the original trigger -- not from the data
  flowing through. n8n executes nodes in connection order. The early node runs, its data is
  available in memory, and the downstream node ignores the flowing data by pulling its parameters
  from the trigger instead.

  Example pattern (Project 2): Signal Log read must execute before Dedup Code. Wire:
  Signal Log Read -> Exa Pull. Exa Pull ignores Signal Log data and retrieves its search parameters
  from the Google Sheets Trigger via cross-node reference. Signal Log rows are then available to
  the Dedup Code node when it runs.

---

## Content Sanitization

Always sanitize raw scraped content before passing to any LLM. Use this pattern in a Code node:

```javascript
let cleaned = ($input.item.json.data || '').toString();
cleaned = cleaned.replace(/\[.*?\]\(https?:\/\/[^\)]+\)/g, '');
cleaned = cleaned.replace(/https?:\/\/\S+/g, '');
cleaned = cleaned.replace(/[\*#]+/g, '');
cleaned = cleaned.replace(/cookie|opt.out|preferences|privacy|GDPR|consent/gi, '');
cleaned = cleaned.replace(/\n+/g, ' ');
cleaned = cleaned.trim();
if (!cleaned || cleaned.length < 50) {
  cleaned = 'No meaningful content available for this company.';
}
cleaned = cleaned.substring(0, 3000);
```

- Cap content at 3000 characters to control token usage
- Always include the empty content fallback -- some pages return nothing useful after sanitization

### Jina AI Content -- Additional Sanitization

When using the Jina AI native node to fetch article content, apply additional cleaning before
the standard sanitization above:

```javascript
// Strip stock ticker tape noise (e.g. "QQQ 717.54 (+0.42%)" patterns on MarketBeat pages)
// Must be applied BEFORE the character cap -- ticker noise fills the budget before article content
cleanText = cleanText.replace(/[A-Z]{1,5}\s+\d+\.\d+\s+\([+-]\d+\.\d+%\)\s*/g, '');
```

---

## Structured Output from LLMs

- Do not use n8n Structured Output Parser nodes -- they inject conflicting schema instructions that cause model output failures
- Instead, instruct the LLM in the prompt to return raw JSON only with no markdown fences
- Add a Code node after the LLM to parse the JSON with a try/catch and regex fallback:
```javascript
const raw = ($json.text || '').trim();
let score = 5;
let rationale = 'Could not parse LLM output.';
try {
  const cleaned = raw.replace(/```(?:json)?\s*/gi, '').replace(/```\s*/g, '').trim();
  const parsed = JSON.parse(cleaned);
  score = Math.min(10, Math.max(1, parseInt(parsed.score, 10)));
  rationale = parsed.rationale || rationale;
} catch (e) {
  const scoreMatch = raw.match(/"score"\s*:\s*(\d+)/);
  const rationaleMatch = raw.match(/"rationale"\s*:\s*"([^"]+)"/);
  if (scoreMatch) score = Math.min(10, Math.max(1, parseInt(scoreMatch[1], 10)));
  if (rationaleMatch) rationale = rationaleMatch[1];
}
return { json: { score, rationale } };
```

- Never rely on $json pass-through after a chainLlm node. The chainLlm node strips all upstream
  item context including arrays. Use `$('NodeName').item.json` in every downstream Code node.
  Treat this as a standing rule: if a chainLlm node is anywhere in the path, $json is dead downstream.

- Every Code node must open by explicitly pulling all needed fields off $json before referencing them.
  A field referenced before assignment causes a ReferenceError with no clear signal of the root cause.

- When rendering an array in a Gmail HTML body expression, always include an Array.isArray guard.
  Without it, if the array field is undefined or malformed, the expression renders `[object Object]`
  instead of failing cleanly.

---

## Merge Node and Synchronization Gates

- Merge node (typeVersion 3) defaults to 2 inputs. Connections wired to index 2 or higher silently
  vanish on import with no error. Always set `"numberInputs"` explicitly in the node parameters to
  match the actual number of incoming connections.

- The correct combineByPosition parameter key is `"combineBy": "combineByPosition"` -- not
  `"combinationMode": "mergeByPosition"`. The wrong key causes silent fallback to Match Fields mode,
  which fails because no matching fields are defined.

- Always enable `"includeUnpaired": true` on Merge nodes used as synchronization gates. Without it,
  the Merge drops items when one branch produces fewer items than expected.

- Pattern -- Merge as synchronization gate (not data aggregator):
  When multiple parallel branches must all complete before a final action fires, use a Merge node
  with one input per branch. The actual payload flows through one input. The other inputs are
  completion signals only. All inputs are necessary for the gate to fire exactly once.
  Example (Project 5): Gmail Welcome (signal), Gmail Manager (signal), Limit (signal), Check_In (data).
  Four inputs. One data carrier. Three signals. Configure with `"mode": "combineByPosition"` and
  `"includeUnpaired": true`.

- Pattern -- Limit node as synchronization reducer:
  Any branch that produces N items and feeds a synchronization Merge gate must have a Limit node
  (maxItems: 1) immediately before the Merge. Without it, the Merge fires N times, producing
  duplicate log rows and duplicate downstream actions.
  Use Limit (not Aggregate) when the downstream node only needs a completion signal, not the data.
  Confirm maxItems is explicitly set and visible in the Parameters panel before exporting -- a Limit
  node with an empty parameters object `{}` defaults to an unspecified limit on import.

- Post-build mandatory check: verify that every node feeding a Merge gate has an entry in the
  connections object. Nodes most likely to have missing connections: Limit, Check_In, and any node
  feeding a multi-input Merge. Missing connections produce no error on import -- the node runs and
  drops its output silently.

---

## IF Node Conditions

- Boolean conditions must use expression mode on the left side: `{{ $json.fieldName }}`
- Use the 'is true' or 'is false' operator -- never 'equal to true' or 'equal to false' -- to avoid type mismatch errors
- IF node conditions do not always import correctly from JSON -- verify and fix manually after import
- The condition left side showing as a static value like "true" instead of an expression is a common import failure
- IF node condition verification is a mandatory post-import check on every build -- it is the single most import-fragile part of any n8n workflow

---

## Gmail and Logging Pattern

Gmail does not pass the email content through to downstream nodes, but it does allow downstream nodes to execute. Use cross-node references in Log nodes to retrieve email body from the Sanitize Text node.

**Preferred pattern -- sequential (confirms delivery before logging):**
```
LLM Chain → Sanitize Text → Gmail → Log → Update Last Contacted
```
- Gmail fires first, confirming delivery
- Log node wired from Gmail's output; uses cross-node ref for email body:
  ```
  {{ $('Sanitize Text Ticket').item.json.text.substring(0, 100) }}
  ```
- Update Last Contacted wired from Log's output

**Alternate pattern -- parallel branch (use when delivery confirmation is not required):**
```
LLM Chain → Gmail          (email branch, terminates)
          → Log → Update   (logging branch, runs in parallel)
```
- Both branches wire from the LLM Chain (or Sanitize Text) node output
- Log uses $json.text directly since it receives output before Gmail
- Do not wire Log from Gmail's output in this pattern

**Rules that apply to both patterns:**
- Never hardcode email addresses in Gmail sendTo fields. Always use dynamic expressions:
  ```
  {{ $('Category Router').item.json.email }}
  ```
  A hardcoded address in sendTo is always a production bug. Add sendTo field verification
  to the pre-export checklist for every workflow.

---

## Deduplication Pattern

- Read the log sheet with a `getRows` node triggered from the same manual trigger (fan-out)
- Use a Code node in runOnceForEachItem mode to check each company against the log
- Pass the log rows as a field on each item (summaryRows array) for downstream checking
- Dedup check example:
```javascript
const { company, summaryRows } = $json;
const cutoff = Date.now() - (30 * 24 * 60 * 60 * 1000);
let isRecent = false;
if (summaryRows && summaryRows.length > 0) {
  isRecent = summaryRows.some(row => {
    const rowCompany = (row.Company || '').trim();
    if (rowCompany.toLowerCase() !== company.toLowerCase()) return false;
    if (!row.Recency) return false;
    return new Date(row.Recency).getTime() > cutoff;
  });
}
return { json: { isRecent, company, ...otherFields } };
```
- Use $now.toISO() for timestamps written to the log sheet

### Set-Based URL Dedup

For "does this URL exist anywhere in the log" checks, do not use the Compare Datasets node.
Compare Datasets performs a JOIN-style positional match -- it pairs items by index, not by searching
the full dataset for a matching value. Use a Code node with a JavaScript Set instead:

```javascript
const logUrls = new Set(
  $('Get Signal Log').all().map(item => item.json['Signal URL'])
);
const items = [];
for (const input of $input.all()) {
  if (!logUrls.has(input.json.url)) {
    items.push({ json: input.json });
  }
}
return items;
```

This is O(1) lookup per item and correctly identifies any URL that exists anywhere in the log
regardless of item order or count.

### Log at Dedup Time, Not End of Pipeline

Articles or records dropped by downstream relevance filters never reach end-of-pipeline append nodes.
If logging only happens at the end, dropped records will be fetched again on the next run and pass
dedup again.

Resolution: Add a Google Sheets Append node immediately after the Dedup Code node. Log every record
that passes dedup with placeholder values for fields not yet determined (e.g. "Not Relevant" for
signal type, summary, classification). Use an Update operation at the end of the pipeline to
overwrite placeholders with real values. Records that never reach the end retain their placeholder
values -- which is useful audit information, not missing data.

### Progressive Data Enrichment Pattern

Two-stage write pattern for any pipeline with a filtering step:

- Stage 1 (at dedup): Append node logs the record immediately with all available fields plus
  "Not Relevant" placeholders for fields that require downstream processing (classifier output,
  scores, inclusion decisions).
- Stage 2 (end of pipeline): Update operation matches on a unique key field (e.g. Signal URL) and
  overwrites placeholder fields with actual values.

"Not Relevant" in classification fields is preferable to blank. Blank is ambiguous -- did the
record not get classified, or did something break? "Not Relevant" is unambiguous.

### Value-Based Pairing After Filter Nodes

Any time a filter node (IF node, dedup Code node) sits between an upstream data source and a
downstream node that needs fields from that source, use value-based lookup instead of index-based
cross-node pairing. Match on a unique field (URL, ID) using `.find()`:

```javascript
const allItems = $('Dedup Code').all();
const match = allItems.find(item => item.json.url === $json.url);
const companyName = match ? match.json.companyName : '';
```

Index-based pairing breaks silently when item counts change after filtering. The error only surfaces
on runs where filtering actually removes items -- making it hard to catch in initial testing where
all items may pass.

### Descending Sort Before Batch Delete

Any workflow that deletes multiple Google Sheets rows sequentially must sort those rows by
row_number descending before the Delete node. Sequential deletion from the top causes index
shifting -- row 5 becomes row 4 after row 4 is deleted, so every other row survives. Bottom-up
deletion eliminates index shifting entirely.

This is a mandatory pattern, not an optimization. Apply it to any cleanup branch that deletes
more than one row.

---

## Exa.ai and Semantic Search APIs

- Exa.ai returns results as a nested `results` array inside the HTTP response body. The parse step
  must loop through the outer array and explode each nested results array into individual items.
  Use a Code node in runOnceForAllItems mode:
  ```javascript
  const items = [];
  for (const response of $input.all()) {
    const results = response.json.results || [];
    const companyName = response.json.companyName || '';
    for (const article of results) {
      items.push({ json: { ...article, companyName } });
    }
  }
  return items;
  ```

- Exa.ai HTTP Request nodes do not automatically carry upstream fields (e.g. companyName) through
  the response. Always add an Edit Fields node immediately after the Exa HTTP Request to re-attach
  any upstream context fields before the parse step. Set the field type to Array when carrying
  a results array -- setting it to String serializes the array and breaks downstream parsing.

- Live semantic search APIs are non-deterministic. The index updates in real time and result
  ranking shifts between calls. Running the pipeline twice minutes apart can return different
  article sets. This is expected behavior and does not indicate a dedup bug.

- Search anchors must disambiguate ambiguous company names. Without disambiguation, "Glean" matches
  articles about gleaning information, "Cursor" matches articles about mouse cursors, and "Elastic"
  matches articles about elastic materials. Append a qualifier to any name that is also a common
  English word (e.g. "Glean AI enterprise", "Cursor AI coding", "Elasticsearch AI"). Use quoted
  phrases to prevent word separation for multi-word names (e.g. "Hugging Face", "Scale AI").

---

## Jina AI Node

- Use the native n8n Jina node, not an HTTP Request to r.jina.ai. Calling Jina via HTTP Request
  triggers DDoS protection blocks when called across multiple companies in sequence. The native node
  handles authentication and rate management cleanly.

- The native Jina node returns data at `$json.data.content` (not `$json.text`). Published time is
  at `$json.data.publishedTime` (not `$json.data.publishedDate`). Article URL is at `$json.data.url`.
  These field names differ from the HTTP Request approach -- always verify actual output structure
  before writing parse code.

---

## Relevance Pre-Filtering

Pre-filter articles or records before any LLM classifier call. Every classifier call costs tokens.
A record that fails relevance still fails classification correctly, but you have spent tokens to
confirm the obvious.

- Title-only filtering is too strict. Legitimate signals sometimes appear in articles where the
  company name does not appear in the headline.
- Full-text filtering is too loose. A single passing mention deep in the body will pass the check
  even when the article is not primarily about the target company.
- Title plus lede (first 500 characters) is the correct balance. Primary subjects are almost always
  named in the opening sentences. Passing mentions tend to appear deeper in the body.

```javascript
const title = ($json.title || '').toLowerCase();
const lede = ($json.text || '').substring(0, 500).toLowerCase();
const isRelevant = companyAliases.some(alias =>
  title.includes(alias.toLowerCase()) || lede.includes(alias.toLowerCase())
);
return { json: { ...$json, isRelevant } };
```

- Alias tables are required for any company with an ambiguous or multi-word name. Aliases must
  cover product names, stock tickers, CEO names, and legal entity names. Build the alias table in
  CLAUDE.md or a config node -- not hardcoded in the filter logic.

---

## Browser Compatibility

- Use Chrome or Edge for n8n -- Firefox has websocket instability issues with the n8n editor
- Save frequently with Ctrl+S during active build and debug sessions

---

## Build Sequence for Token Management

- Break large workflow builds into phases to stay within Claude Code token limits
- Recommended phases:
  - Phase 1: Trigger, data reading, dedup logic
  - Phase 2: Enrichment (scraping, APIs, sanitization), first LLM
  - Phase 3: Scoring LLM, routing IF node
  - Phase 4: Output nodes (email, logging)
- End each phase prompt with an explicit instruction to stop and wait
- After each phase, retrieve the JSON from the GitHub branch before starting the next phase

---

## Post-Import Checklist

After importing any Claude Code-generated workflow into n8n, verify in this order:

1. Every IF node condition -- confirm the left side is in expression mode and the operator is correct
2. All LLM prompt fields are wrapped in {{ }} and expressions are evaluating
3. Every LLM Chain node is followed by a Sanitize Text Code node before any Gmail node
4. Log nodes that run after Gmail use cross-node refs (e.g. `$('Sanitize Text Ticket').item.json.text`) -- not `$json.text`
5. Trigger node typeVersion matches the n8n Cloud instance
6. All placeholder Sheet IDs, Task List IDs, and email addresses are filled in
7. Credential names match exactly what is in n8n's credential manager
8. Merge node is set to combineByPosition (not Matching Fields) and includeUnpaired is enabled
9. Every node has an outgoing connection in the connections object -- Limit, Check_In, and Merge
   gate inputs are the most commonly missing
10. Google Tasks due date fields use full ISO 8601 format (`2026-07-01T00:00:00.000Z`), not bare date strings
11. Status sheet column mappings pull from the Build Status Row node, not from the original trigger
12. sendTo fields in all Gmail nodes are dynamic expressions, never hardcoded addresses
13. Any pipeline with a cleanup delete branch: confirm rows are sorted by row_number descending before the Delete node
14. Jina AI calls use the native n8n Jina node, not HTTP Request to r.jina.ai
15. Any funding amount or suffixed numeric comparison uses parsed values, not raw strings

---

## Pre-Export PII Checklist

Run this check before exporting any workflow JSON to GitHub or for public sharing.
Replace all literal values with clearly labeled placeholders (e.g. `YOUR_GOOGLE_SHEET_ID`).

- sendTo fields in Gmail nodes -- must be dynamic expressions, never hardcoded addresses
- Google Sheet IDs and URLs in documentId and sheetName fields
- Task List IDs in Google Tasks nodes
- OAuth2 credential IDs in all credentials blocks
- Webhook IDs in trigger nodes
- n8n instance ID in the meta block
- Workflow ID and versionId at the root level
