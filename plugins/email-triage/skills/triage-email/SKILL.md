---
name: triage-email
description: >
  Pull unread support emails from the last 24 hours, generate a draft reply for each,
  and present them in an interactive HTML UI for the human to review, copy, and send.
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

Pull unread emails from the last 24 hours, generate a draft reply for each, and
present them in an interactive HTML UI — ready for a human to review, copy, and send.

**Important:** This skill does NOT create drafts in the email provider, does NOT send
emails, and does NOT push anything back to the inbox. The only output is the HTML UI
(or plain text fallback). The human copies drafts into their email client manually.

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

If not, check if there is Context Link skills already installed, namely get-context and ask-question

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
Retrieving lessons from Context Link -> customer-support-email-lessons
```

Use the **get-context** skill with the slug `customer-support-email-lessons` (no mode
param needed). If lessons are returned, hold them in memory and pass them as context
when drafting replies in Step 2. These lessons take priority over general style rules
in draft-email-response — they represent real corrections from past edits.

If the fetch returns empty or fails, continue without lessons.

**Important:** the lessons file is meta-rules about *how* to reply (tone, sign-off,
structure, common phrasing pitfalls). It is NOT the product knowledge itself.
Loading lessons does not substitute for the per-email Context Link lookup in Step 2.

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
- `priority` — `"urgent"` or `"normal"` (see "Detecting priority" below)

Follow the provider-specific reference file for exact tool calls / API requests.

#### Detecting priority

Mark an email as `priority: "urgent"` if any of the following apply. Otherwise treat it
as `"normal"`. Be conservative — over-flagging dilutes the signal.

- **Subject signals:** the subject contains "URGENT", "ASAP", "Emergency", "Help!!",
  multiple exclamation marks, "down", "broken", or "not working" (case-insensitive).
- **Provider flags:** the message is flagged as high-importance / priority by the
  email provider (e.g. Zoho `priority: "high"`, Gmail importance markers).
- **Time-pressured payment / refund issues:** body explicitly references being charged
  in error, a refund needed "immediately", a failed launch in progress, or a live
  customer-facing pre-order being broken right now.
- **Live-store breakage from an existing customer:** explicit mention that something
  is currently broken on a live storefront (e.g. "checkout is failing", "site is
  showing the wrong button", "launch goes live in X hours").
- **Repeat-bump within the same thread:** the customer has sent two or more emails
  in the same thread within the last few hours chasing for a response.

Do NOT mark prospect demo requests or general questions as urgent, even if phrased
enthusiastically. Urgency is reserved for time-pressured operational issues.

After fetching, present a summary table to the user (urgent rows flagged):

```
Found {N} unread emails from the last 24 hours:

| # | From | Subject | Received | Priority |
|---|------|---------|----------|----------|
| 1 | customer@example.com | URGENT! Pre-order broken | 30m ago | 🟧 Urgent |
| 2 | ... | ... | ... | — |
```

**Do not ask for confirmation.** Proceed to draft replies for all emails automatically.

The only exceptions — skip (and flag to the user) emails that appear to be:
- Spam or marketing newsletters
- Phishing or suspicious/malicious
- Automated system notifications that don't need a reply

For skipped emails, note them at the end: `Skipped {N} emails (spam/automated).`

### Step 2 — Look up product context, THEN draft

This step has two phases. **Phase 2a is mandatory for every non-trivial email.**
Skipping it produces confidently-worded wrong drafts on factual topics, which is
worse than no draft at all.

#### Phase 2a — Classify each email, then look up context

For each email, decide whether it is **trivial** or **non-trivial**.

**Trivial emails** (skip Context Link lookup, go straight to 2b):

- Pure acknowledgement: "thanks for the URLs, I'll take a look"
- Scheduling: confirming or declining a time
- Asking the customer for more info you don't yet have ("which store is this on?")
- Routing: "I'll loop in X" / "let me check with the team"
- Replies that contain no factual claim about how anything works

**Non-trivial emails — Context Link lookup is MANDATORY before drafting:**

- Any explanation of how PreProduct works (charging, refunds, fulfilment holds,
  redirects, templates, listing manager, automations, payment plans, deposits,
  pay-early, isolated cart/checkout, customer cancellation, etc.)
- Any explanation of how Shopify or another platform interacts with PreProduct
  (B2B catalogues, Shop Pay Installments, Affirm, payment methods, sales channels,
  theme behaviour, split shipping, mixed carts, etc.)
- Any feature-availability question ("does X work with Y?")
- Any pricing or plan question
- Any bug report — the right answer is often a platform limitation, not a bug,
  and Context Link is where the canonical framing lives
- Any "why did X happen on order Y" question
- Anything you are about to write a confident-sounding paragraph about

**Default is non-trivial.** If you are unsure which bucket an email falls into,
treat it as non-trivial and run the lookup. The cost of an extra Context Link query
is trivial. The cost of an invented answer is a customer email the user has to
rewrite from scratch — and a wrong claim sent to a real customer if they miss it.

For each non-trivial email:

1. **Identify the 1–3 distinct topics or factual claims the reply needs to verify.**
   Write them down explicitly before you fetch. Example for an email about a mixed-SKU
   charge: "(a) how does charging behave when only some line items on an order are
   being charged? (b) is per-line-item charging a Shopify Plus feature?"
2. **For each topic, run one of:**
   - `get-context` with a specific slug if you know which slug covers the topic
     (e.g. `charging-mechanics`, `refund-behaviour`, `mixed-cart-shipping`).
   - `ask-question` with the question phrased plainly if you don't know the slug.
     This searches the whole Context Link knowledge base semantically. Prefer this
     when in doubt — guessing a slug that doesn't exist returns nothing useful, but
     `ask-question` will find the right material.
3. **Hold the returned content per topic. Do not draft yet.**

Record what you queried for each email — surface the queries in the final summary so
the human can see whether you reached for the right context. Format suggestion:

```
| # | From | Subject | Priority | Context queries |
|---|------|---------|----------|-----------------|
| 1 | kirill@... | Re: Urgent charging | 🟧 | charging-mechanics, refund-behaviour |
| 2 | shoko@... | Re: Help with PreProduct | — | (trivial — ack only) |
```

#### Phase 2b — Draft each reply

Once Context Link content is in hand (or you've classified the email as trivial),
draft the reply.

**Drafting rules:**

- **Lead with what Context Link said, not with internal knowledge.** If Context Link
  says "this is a Shopify platform limitation, here's the doc", your first line
  should reflect that. Do NOT open with "that sounds off, let me dig in" if the
  canonical answer is a known platform constraint that's already documented.
- **If Context Link returned nothing useful on a topic, say so honestly in the draft**
  ("I want to double-check this on our side before answering definitively") rather
  than filling the gap with guesses. A shorter, honest draft beats a longer
  speculative one.
- **Apply the lessons loaded in Step 0** for tone, structure, sign-off, anchor-text
  style, and the recurring failure modes they call out.

If the **draft-email-response** skill is available, you MAY delegate the actual
prose generation to it — passing along the Context Link content you just retrieved
in Phase 2a. But do NOT rely on draft-email-response to do the lookups itself: this
skill is responsible for ensuring the lookups happen in Phase 2a, full stop. If
draft-email-response runs its own `get-context` calls on top, that's additive, not
a substitute. The point is that no non-trivial email reaches the drafting stage
without canonical product context in hand.

**If draft-email-response is not available**, draft inline using the rules above
plus any lessons loaded in Step 0. Fall back to the placeholder format only as a
last resort:

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

**Order the array urgent-first, then chronological.** Within each tier, sort by
`received_at` ascending (oldest first):

1. All emails with `priority: "urgent"`, oldest first.
2. All other emails (`priority: "normal"` or omitted), oldest first.

Urgent emails render with an orange left accent on the card and an "Urgent" pill in
the meta line — the template handles the styling automatically when you set the
`priority` field.

```js
const DRAFTS = [
  {
    id: 1,
    priority: "urgent",            // omit or set "normal" for non-urgent
    from: "{fromAddress}",
    subject: "{subject}",
    received: "{relative time, e.g. '~6h ago'}",
    draft: "{scrubbed draft reply text from Step 2}",
    actions: [
      "{required action 1}",
      "{required action 2, if any}"
    ]
  },
  // ... urgent items first (oldest-first within urgent), then normal items oldest-first
];
```

The UI renders as interactive collapsible cards:
- **Checkbox** on the left of each card — tick to mark as done (fades and strikes through)
- **Orange left accent + "Urgent" pill** on cards where `priority === "urgent"`
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

- **If Cowork tools are found** -> write the populated `.html` file to the user's
  workspace folder and present it as an artifact. It renders inline in Cowork.

- **If no Cowork tools** (Claude Code / CLI) -> write the populated `.html` file to a
  temp path and tell the user where to find it:
  ```
  Done — {N} draft replies ready. Open this file in your browser to review:
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
- If a Context Link query in Phase 2a returns empty or errors out for a non-trivial
  email, do NOT fall back to internal knowledge. Either re-phrase the query and try
  once more, or proceed to draft with an explicit "I want to double-check this on our
  side" hedge so the user can see the gap and either fill it in or rewrite.

---

## Dependencies

This skill relies on the following other skills:

- **get-context** — Retrieves internal knowledge from Context Link via a simple
  WebFetch GET against a known slug. Used in Step 0 (lessons) and Step 2 (per-email
  product context). If not available, the skill degrades gracefully but the user
  must be told that drafts may be missing canonical product framing.
- **ask-question** — Semantic search across the whole Context Link knowledge base.
  Preferred over `get-context` in Phase 2a when the right slug is unknown.
- **draft-email-response** — Optional. Generates context-aware draft replies given
  Context Link content already in hand. If not available, this skill drafts inline
  using the lessons + Context Link content directly. Located in the same plugin at
  `skills/draft-email-response/`.
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
   When a factual correction surfaces, also consider whether the underlying topic
   deserves its own dedicated Context Link slug — if so, flag that to the user and
   offer to create it.

2. **Save the lessons.** How you save depends on the knowledge mode:

   **If using Context Link (default):**
   Use the **get-context** skill with the slug `customer-support-email-lessons` to
   retrieve existing lessons. Then merge the new lessons in:
   - If a new lesson overlaps with an existing one, update the existing one rather
     than adding a near-duplicate.
   - Never drop or overwrite existing lessons. Only add or refine.
   - Keep the list concise and context-window-efficient.
   Use the **update-memory** skill with the slug `customer-support-email-lessons`
   to save the merged list.
   Confirm: `Recorded {N} new lesson(s) to customer-support-email-lessons on Context Link.`

   **If using Claude memory fallback (`knowledge_mode = "claude-memory"`):**
   Save lessons to a file called `email_support_lessons.md` in your auto memory
   directory. If the file already exists, read it first and merge new lessons in —
   deduplicate and condense the same way. Add or update a pointer in `MEMORY.md`
   so the lessons are discoverable in future sessions.
   Confirm: `Recorded {N} new lesson(s) to Claude memory.`

---

## Important notes

- Never send an email automatically. Never create drafts in the email provider.
- The only output is the HTML UI (or plain text fallback). The human copies drafts manually.
- Draft all emails automatically — do not ask for confirmation. Only skip spam, phishing, or automated notifications.
- Respect the user's choice if they want to skip specific emails after seeing the summary.
- The `[REPLY NEEDED]` tag is a convention — the user can customise it.
- Do NOT use sub-agents (Agent tool) anywhere in this skill. All work runs inline.
  Sub-agents cannot reliably access WebFetch or MCP tools in Cowork scheduled tasks.
- Context Link calls use the **get-context** skill (or **ask-question** for semantic
  search), which do simple WebFetch requests. Never use browser navigation (Chrome)
  for Context Link — it's just a URL fetch.
- **Per-email Context Link lookup is non-negotiable for non-trivial emails.** Loading
  the lessons file in Step 0 does NOT substitute for it — lessons are meta-rules
  about how to reply, not the product knowledge itself. If you find yourself about
  to write a confident paragraph about charging, refunds, fulfilment, Shopify
  compatibility, pricing, or any other product mechanic without having run a
  Context Link query for that specific topic in this triage run, stop and run the
  query first.
