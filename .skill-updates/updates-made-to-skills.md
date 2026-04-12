# Updates Made to Email Triage Skills

Date: April 12, 2026

---

## Overview

Two changes were made to improve draft quality and prevent lesson data loss. Both use sub-agents to isolate complex tasks from the main triage flow.

---

## Change 1: Customer history lookup before drafting

**Files changed:** `triage-email/SKILL.md`, `draft-email-response/SKILL.md`

**Problem:** Drafts had no awareness of previous conversations with the customer. Every reply treated the sender as a stranger, even if there was an ongoing thread or established rapport.

**Solution:** Step 2 now splits into two sub-steps:

- **Step 2a** launches a sub-agent per unique sender that queries Context Link with the sender's email address. The sub-agent returns a concise summary (under 100 words) covering previous conversations, issues discussed, and what's resolved vs. still open. Multiple senders are looked up in parallel.

- **Step 2b** passes that `customer_history` summary into `draft-email-response` as a new arg. The drafting skill uses it to tailor the reply: referencing prior conversations, avoiding re-explanations, and matching established rapport.

**Changes in `draft-email-response/SKILL.md`:**
- Added `customer_history` arg to frontmatter
- Added guidance in Step 5 (Draft the reply) on how to use the history when composing

---

## Change 2: Sub-agent for lesson merging

**Files changed:** `triage-email/SKILL.md`

**Problem:** The main agent was merging new lessons into the existing Context Link list directly. Because the main agent's context window was already full from the triage run, existing lessons were sometimes overwritten instead of properly merged.

**Solution:** The "Learning from edits" section now delegates lesson saving to a sub-agent. The main agent formulates the new lessons from diffs/instructions, then passes them to a sub-agent that:

1. GETs the full existing lesson list from Context Link (`customer-support-email-lessons`)
2. Reads every existing lesson carefully in its own clean context
3. Merges new lessons in without dropping or overwriting existing ones
4. POSTs the merged list back via `update-memory`
5. Reports how many lessons were added or updated

This ensures the sub-agent has full context for a proper merge, since it only needs to focus on the lesson list rather than juggling the entire triage session.

---

## How to deploy

These files need to replace their counterparts in the `email-triage` plugin source:

- `updated-skills/triage-email/SKILL.md` replaces `skills/triage-email/SKILL.md`
- `updated-skills/draft-email-response/SKILL.md` replaces `skills/draft-email-response/SKILL.md`

The plugin is currently installed from the `email-triage-plugin` marketplace, so the source needs to be updated there.
