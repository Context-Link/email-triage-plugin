---
name: triage-email
description: >
  Pull unread support emails from the last 24 hours, generate a draft reply for each,
  and push those drafts back into the correct thread — ready for a human to review and send.
  Supports Gmail (via MCP connector) and Zoho Mail (via MCP connector).
  Use this skill whenever the user says "triage emails", "triage inbox", "draft replies",
  "process support emails", "check support inbox", or any variation of pulling emails
  and creating draft responses. Also trigger when the user mentions triaging, batching,
  or bulk-replying to customer emails.
args:
  email_provider:
    description: "The email provider to use: 'gmail' or 'zoho'"
    required: false
  email_address:
    description: "The specific email address to triage (e.g. hello@example.com). Use 'all' for all addresses."
    required: false
  output:
    description: "Output format: 'ui' (default, interactive HTML) or 'chat' (plain text fallback). UI is auto-delivered as Cowork artifact or opened in browser for Claude Code."
    required: false
    default: "ui"
---

# Triage Email Skill

Pull unread emails from the last 24 hours, generate a draft reply for each, and push
those drafts back as threaded replies — ready for a human to review and send.

---

## Pre-flight: Detect the email provider

Check if `email_provider` was passed as an argument. If so, use it directly.
If not, ask the user which provider to use.

The two supported providers are **Gmail** and **Zoho Mail**.

| Provider | Integration method | Reference file |
|----------|-------------------|----------------|
| Gmail (`gmail`) | MCP tools (discovered via `ToolSearch`) | `gmail.md` (same directory as this file) |
| Zoho Mail (`zoho`) | MCP tools (discovered via `ToolSearch`) | `zoho.md` (same directory as this file) |

Read the appropriate reference file before starting. Only ask if `email_provider`
was not provided:

> Which inbox should I pull from — **Gmail** or **Zoho**?

## Pre-flight: Check knowledge base

Check if there's a Context Link URL configured below. If there is one, use it.

Context Link: 

If no Context Link is configured above:
1. Ask the user: "Do you use [Context Link](https://context-link.ai) for your knowledge base? If so, paste your link and I'll save it for next time. If not, I can try to use Claude's built-in memory instead. It's more limited but works without any setup."
2. If the user provides a Context Link URL, paste it after `Context Link:` above for future use.
3. If the user says they don't use Context Link, set `knowledge_mode = "claude-memory"` for this run. In this mode:
   - Skip the get-context skill entirely (no external knowledge retrieval)
   - Use Claude's own memory system to load and save lessons (see fallback steps below)

## Pre-flight: Choose the email address

Check if `email_address` was passed as an argument. If so, use it directly
(skip the address selection prompt). If the value is `"all"`, skip address filtering.

If `email_address` was not provided, fetch the available email addresses/aliases
from the account and present them to the user:

> I found these email addresses on your account:
>
> 1. **hello@example.com** (primary)
> 2. **admin@example.com** (alias)
> 3. **support@example.com** (alias)
>
> Which address should I triage? Or should I pull unread emails for all of them?

Use the selected address to filter results in Step 1 — only show emails sent **to**
that address. If the user picks "all", skip the filter and show everything.

See the provider-specific reference file for how to filter by recipient address.

---

## The workflow

### Step 0 — Load lessons

Before processing any emails, fetch previously recorded support lessons once.

**If using Context Link (default):**

```
🔗 Retrieving lessons from Context Link → customer-support-email-lessons
```

Use the **get-context** skill with the slug `customer-support-email-lessons` (no mode
param needed). If lessons are returned, hold them in memory and pass them as context
when drafting replies in Step 2. These lessons take priority over general style rules
in draft-email-response — they represent real corrections from past edits.

If the fetch returns empty or fails, continue without lessons.

**If using Claude memory fallback (`knowledge_mode = "claude-memory"`):**

Check your auto memory directory for a file called `email_support_lessons.md`. If it
exists, read it and load the lessons. Apply them the same way — they take priority over
general style rules. If the file doesn't exist yet, continue without lessons.

### Step 1 — Fetch unread emails from the last 24 hours

Pull all unread messages received in the last 24 hours. For each message, capture:

- `message_id` — unique ID for the message
- `thread_id` — thread/conversation ID (for threading the reply correctly)
- `from` — sender email address
- `subject` — email subject line
- `body` — full email body (plain text preferred, HTML fallback)
- `received_at` — timestamp

Follow the provider-specific reference file for exact tool calls / API requests.

After fetching, present a summary table to the user:

```
Found {N} unread emails from the last 24 hours:

| # | From | Subject | Received |
|---|------|---------|----------|
| 1 | customer@example.com | Order not received | 2h ago |
| 2 | ... | ... | ... |
```

**Do not ask for confirmation.** Proceed to draft replies for all emails automatically.

The only exceptions — skip (and flag to the user) emails that appear to be:
- Spam or marketing newsletters
- Phishing or suspicious/malicious
- Automated system notifications that don't need a reply

For skipped emails, note them at the end: `⚠ Skipped {N} emails (spam/automated).`

### Step 2 — Generate reply using draft-email-response

For each email from Step 1, gather customer history and then draft a reply.

#### Step 2a — Look up customer history (sub-agent)

Before drafting, launch a **sub-agent** to fetch the sender's previous email history
from Context Link. This runs concurrently with the topic-based context lookup that
draft-email-response does internally, so it adds customer awareness without slowing
things down.

```
Agent(
  prompt: "Use the get-context skill to look up '{fromAddress}' on Context Link.
           Return a concise summary (under 100 words) covering:
           - How many previous conversations exist
           - What issues were discussed
           - What was resolved and what's still open
           - Any relevant context for the current email: '{subject}'
           If no history is found, just say 'No previous history found.'",
  description: "Customer history lookup"
)
```

Hold the returned summary for use in Step 2b. If the sub-agent returns no history
or fails, continue without it — the draft will still work, just without customer
context.

**Parallelism note:** When processing multiple emails, you can launch all customer
history sub-agents at the same time (one per unique sender). No need to wait for
one to finish before starting the next. Deduplicate by sender — if two emails are
from the same address, reuse the same history for both.

#### Step 2b — Draft the reply

Use the **draft-email-response** skill to generate the reply. Pass the customer
history from Step 2a as additional context:

```
draft-email-response(
  from: "{fromAddress}",
  subject: "{subject}",
  body: "{summary or body content}",
  customer_history: "{summary from Step 2a, or empty if none}",
  prompt_lessons: "false",
  get_lessons: "false"
)
```

The draft-email-response skill will:
1. Identify the core issue in the email
2. Look up relevant product/feature context via the **get-context** skill (Context Link)
3. Use the customer history to tailor the reply (e.g. reference prior conversations,
   avoid re-explaining things already discussed, acknowledge ongoing issues)
4. Draft a warm, clear, practical reply

Collect the draft reply text returned by the skill for each email.

**If draft-email-response is not available**, fall back to the placeholder format:

```
Subject: Re: {original_subject}
Body:    [REPLY NEEDED] — This email requires a human reply.

--- Original message from {from} ---
{first 3 lines of body}
```

The placeholder tag `[REPLY NEEDED]` makes these easy to find and filter in the inbox.

### Step 3 — Display the drafts

After all drafts are generated, present them using the **draft display UI**.

Read the HTML template at `draft-display.html` (same directory as this file). Copy its
full contents, then replace the `DRAFTS` array with the real data from Steps 1 and 2:

**Order the array oldest-first** (earliest `received_at` first) so the user works
through emails in chronological order.

```js
const DRAFTS = [
  {
    id: 1,
    from: "{fromAddress}",
    subject: "{subject}",
    received: "{relative time, e.g. '~6h ago'}",
    draft: "{scrubbed draft reply text from Step 2}",
    actions: [
      "{required action 1}",
      "{required action 2, if any}"
    ]
  },
  // ... one entry per email, oldest first
];
```

The UI renders as interactive collapsible cards:
- **Checkbox** on the left of each card — tick to mark as done (fades and strikes through)
- Click to expand/collapse and see the full draft with a **Copy draft** button
- **Required actions** appear as checkboxes so the rep can tick them off
- "Expand all / Collapse all" toggle at the top

**Important:** Do not use the example data from the template. Replace the `DRAFTS` array
entirely with real drafts from the triage run. Escape any quotes or backslashes in the
draft text so the JS string stays valid.

#### Delivery: Cowork vs Claude Code

**Auto-detect the environment** by checking if `mcp__cowork__` tools are available:

```
ToolSearch(query: "+cowork", max_results: 3)
```

- **If Cowork tools are found** → write the populated `.html` file to the user's
  workspace folder and present it as an artifact. It renders inline in Cowork.

- **If no Cowork tools** (Claude Code / CLI) → write the populated `.html` file to a
  temp path and tell the user where to find it:
  ```
  ✓ {N} draft replies ready. Open this file in your browser to review:
    file:///tmp/email-drafts-{YYYY-MM-DD}.html
  ```
  Then ask if they'd like you to open it automatically:
  ```bash
  open /tmp/email-drafts-{YYYY-MM-DD}.html      # macOS
  xdg-open /tmp/email-drafts-{YYYY-MM-DD}.html  # Linux
  ```

Both environments use the exact same HTML file — no build step, no dependencies.

---

## Error handling

- If no unread emails are found, tell the user: "No unread emails from the last 24 hours — inbox zero!"
- If a specific draft fails to create, log the error, skip it, and continue with the rest. Report failures at the end.
- If the email provider connection fails (e.g. Gmail MCP not connected, Zoho MCP not connected), give the user clear instructions on how to fix it. For Zoho, direct them to [zoho.com/mcp](https://www.zoho.com/mcp/) to set up the connector.
- If artifact rendering fails for any reason, fall back to plain text chat output.

---

## Dependencies

This skill relies on two other skills:

- **draft-email-response** — Generates context-aware draft replies (Step 2b).
  Located in the same plugin at `skills/draft-email-response/`. If not available,
  the skill falls back to placeholder `[REPLY NEEDED]` drafts.
- **get-context** — Retrieves internal knowledge from Context Link. Used by
  draft-email-response to look up product docs, support history, and policies.
  Also used by the customer history sub-agent (Step 2a) to look up previous
  conversations by sender email address.
  If not available, draft-email-response can still generate replies but without
  internal context.
- **update-memory** — Saves lessons learned from the user's email edits to the
  `customer-support-email-lessons` namespace on Context Link. If not available
  and the user is not using Claude memory fallback, the lesson-learning step is
  skipped silently.

Assets:
- **draft-display.html** — Self-contained HTML/CSS/JS template for rendering draft
  replies as interactive collapsible cards. Works in both Cowork (as artifact) and
  Claude Code (opened in browser). Located in the same directory as this file.

---

## Learning from edits

After all drafts have been delivered, prompt the user once (not per email):

> If you edited any of the drafts before sending, paste in the responses you actually
> sent — or just tell me what to do differently next time — so I can record lessons
> for future.

The user may respond in one of three ways:

**A) They paste one or more edited replies.** Diff each draft against what they sent.
Identify meaningful changes — tone shifts, factual corrections, structural changes,
things cut or added. Formulate concise, actionable lessons from the diffs.

**B) They give direct instructions** (e.g. "don't do X", "always do Y", "shorter intros").
Treat these as lessons directly — no diffing needed.

**C) They say they used drafts as-is or decline.** Move on — no lessons to record.

For options A and B:

1. **Formulate concise, actionable lessons.** One or two sentences each. Can cover
   tone, facts, preferred phrasing, when to request access, what to cut, etc.

2. **Save the lessons via sub-agent.** Delegate the save to a sub-agent so it has
   full context to retrieve, merge, and write back without losing existing lessons.

   **If using Context Link (default):**

   ```
   Agent(
     prompt: "You need to merge new support email lessons into the existing
              lesson list on Context Link.

              NEW LESSONS TO ADD:
              {formatted list of new lessons from the diffs/instructions}

              Steps:
              1. Use the get-context skill to retrieve the current lessons
                 from the slug 'customer-support-email-lessons'.
              2. Read every existing lesson carefully.
              3. Merge the new lessons into the existing list:
                 - If a new lesson overlaps with an existing one, update the
                   existing one rather than adding a near-duplicate.
                 - Never drop or overwrite existing lessons. Only add or refine.
                 - Keep the list concise and context-window-efficient.
              4. Use the update-memory skill with the slug
                 'customer-support-email-lessons' to save the merged list.
              5. Report back how many lessons were added or updated.",
     description: "Merge support email lessons"
   )
   ```

   After the sub-agent completes, confirm:
   `✓ Recorded {N} new lesson(s) to customer-support-email-lessons on Context Link.`

   **If using Claude memory fallback (`knowledge_mode = "claude-memory"`):**
   Save lessons to a file called `email_support_lessons.md` in your auto memory
   directory. If the file already exists, read it first and merge new lessons in —
   deduplicate and condense the same way. Add or update a pointer in `MEMORY.md`
   so the lessons are discoverable in future sessions.
   - Confirm: `✓ Recorded {N} new lesson(s) to Claude memory.`

---

## Important notes

- Never send an email automatically. Only create drafts.
- Draft all emails automatically — do not ask for confirmation. Only skip spam, phishing, or automated notifications.
- Respect the user's choice if they want to skip specific emails after seeing the summary.
- The `[REPLY NEEDED]` tag is a convention — the user can customise it.
