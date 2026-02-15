---
name: email-triage
description: >
  Reads the user's Gmail inbox and classifies every email as Suspected Junk,
  Action Needed, Informational, or Unknown based on a user-provided
  triage-rules.md file. Then applies Gmail labels and archives junk via Chrome
  browser automation.

  Use this skill whenever the user says "triage my email", "label my email",
  "sort my email", "clean up my inbox", "process my email", or anything that
  sounds like they want their inbox organized, filtered, or prioritized. Also
  trigger if the user mentions email labeling, email archiving, or inbox zero
  workflows.
license: MIT
---

# Email Triage

Scan a Gmail inbox, classify every message using the user's own rules, apply
Gmail labels, and archive suspected junk — all in one pass.

## Overview

This skill connects two tools together: the Gmail MCP connector (for reading
emails) and Chrome browser automation (for applying labels and archiving in
Gmail's web UI). The user supplies a `triage-rules.md` file that defines which
emails matter and which don't. The skill reads those rules, scans the inbox,
classifies each email, and then acts on the results.

**Known limitation — labeling requires browser automation.** The Gmail MCP
connector is read-only (search, read message, read thread, read profile). It
does not support `users.messages.modify` or any write operations, so labels
must be applied via Chrome browser automation. This is slower and less reliable
than an API call. If a Gmail connector with write access (label/modify/archive)
becomes available in the future, update Step 5 to use it instead of browser
automation.

The whole flow is designed around transparency. Every "Suspected Junk" and
"Unknown" classification gets logged with reasoning so the user can audit
decisions and correct mistakes. Over time this feedback loop makes the rules
more accurate.

---

## Step 0 — Check connectors

Before doing anything, confirm that both required connectors are working.

**Gmail MCP connector**
Try calling `search_gmail_messages` with a simple query like `in:inbox`. If the
tool is not available or errors out, tell the user:

> The Gmail connector isn't connected. Please add it from the connectors menu
> so I can read your inbox.

**Chrome browser automation**
Try calling `tabs_context_mcp` to see if the browser extension is responding.
If it fails or returns nothing, tell the user:

> Chrome browser automation isn't responding. Try restarting the Claude in
> Chrome extension (or restarting Chrome entirely), then reconnect.

Do not proceed until both connectors are confirmed working.

---

## Step 1 — Load the rules

Read `triage-rules.md` from the user's working folder.

If the file is missing, tell the user and point them to the template:

> I couldn't find `triage-rules.md` in your folder. This file tells me how to
> classify your emails. I've included a template at
> `reference/triage-rules-template.md` inside this skill — you can copy it to
> your working folder, customize it with your own rules, and then ask me to
> triage again.

Then read the template from `references/triage-rules-template.md` and show
its contents so the user can see what to fill in.

Once you have the rules file, parse it carefully. Pay attention to:

- The **mode** setting (review vs auto) — this controls whether you present
  results for approval or act immediately.
- Each **rule** with its signals and exceptions.
- The **Trusted Domains** list — senders from these domains are almost always
  kept.
- The **Trusted Sources** list — specific senders or newsletters to keep.
- The **Decision Framework** priority order — follow it exactly as written.

---

## Step 2 — Scan the inbox

Use `search_gmail_messages` with a query that excludes already-labeled emails:

```
in:inbox -label:Suspected-Junk -label:Action-Needed -label:Informational
```

This filters out emails that were classified in a previous session so you only
see new, unprocessed messages. (Gmail label names with spaces become hyphenated
in search queries.)

The search results include sender, subject, and a snippet for each message.
That's usually enough to classify an email. Only call `read_gmail_message` for
the full body when classification is genuinely ambiguous from the metadata
alone — for example, when the subject is vague and the snippet doesn't contain
enough signal to match any rule confidently.

### Batch size

Process up to **25 emails per session**. If the inbox has more than 25
unprocessed emails, work through the first 25, complete all steps (classify,
label, log), and then ask the user:

> There are more unprocessed emails in your inbox. Want me to process the next
> batch?

This keeps each session manageable and gives the user a natural checkpoint to
review results, correct mistakes, or stop. If the user says yes, fetch the
next page and repeat the full workflow.

---

## Step 3 — Classify each email

For every email that wasn't skipped, run it through the decision framework
from the user's `triage-rules.md`. The framework typically works as a priority
chain — check each condition in order and stop at the first match.

Assign one of these labels:

| Label | Meaning |
|---|---|
| **Suspected Junk** | Email the user doesn't want to see. Matches an archive rule and no exception applies. |
| **Action Needed** | Email the user needs to respond to or take action on. |
| **Informational** | Worth reading, but no response or action required. |
| **Unknown** | Doesn't clearly match any rule. When in doubt, classify here rather than guessing wrong. |

For each email, note which rule matched (or "no match") and a brief reason why.
You'll need this for the summary and the log.

**Key principle:** False negatives are better than false positives. If you're
unsure whether something is junk, classify it as Unknown — not Suspected Junk.
The user can always reclassify, but accidentally archiving an important email
is much worse than leaving a junk email in the inbox.

---

## Step 4 — Summarize results

How you present results depends on the mode setting in the rules file.

### Review mode

Present a summary table with your proposed classification for every email.
Include these columns:

| From | Subject | Label | Rule | Reasoning |
|---|---|---|---|---|

The user can then:
- **Approve all** — proceed to labeling.
- **Reject specific classifications** — you'll reclassify those as the user
  directs.
- **Ask questions** — explain your reasoning for any email.

Do not proceed to labeling until the user gives the go-ahead.

### Auto mode

Skip the approval step. Instead, present a brief summary after labeling is
complete:

> Processed **N** emails:
> - Suspected Junk: X
> - Action Needed: Y
> - Informational: Z
> - Unknown: W

---

## Step 5 — Label and archive via Chrome

This is the browser automation step. Gmail doesn't have an API for labeling
through the MCP connector, so you'll drive the Gmail web UI directly.

The fastest approach is to batch emails by label — select all emails that share
the same classification and apply the label in one operation. Fall back to
one-at-a-time processing only when batch selection isn't working reliably.

### Setup

1. Call `tabs_context_mcp` to get available tabs.
2. Navigate to `https://mail.google.com` if not already there. Use an existing
   tab if one is already on Gmail; otherwise create a new tab.
3. Wait for Gmail to fully load. Gmail can be slow — use `read_page` to confirm
   the inbox has rendered before proceeding.
4. **Create labels up front.** Before processing any emails, ensure all three
   labels exist in Gmail: "Suspected Junk", "Action Needed", "Informational".
   Navigate to the label manager (Settings gear → Labels, or use the label
   picker's "Create new" option) and create any that are missing. Doing this
   once at the start avoids interrupting the batch flow later.

### Batch labeling (preferred approach)

Group your classified emails by label. Then for each label group:

1. **Search for the batch.** In Gmail's search bar, search for the first email
   in the group by subject or sender. Use `find` to locate the search input.
2. **Select emails.** Click the checkbox next to each matching email in the
   results list. You can select multiple emails before applying a label. If the
   group has more emails than fit on one search results page, process in
   sub-batches.
3. **Apply the label.** With all target emails selected, click the Labels button
   (tag/label icon) in the toolbar. Search for or check the appropriate label
   in the picker and apply it.
4. **Archive if Suspected Junk.** If you're processing the Suspected Junk
   batch, click the Archive button after labeling while the emails are still
   selected. This removes them from the inbox in one action.
5. **Confirm success.** Take a screenshot to verify the label was applied and
   (for Suspected Junk) the emails left the inbox.
6. **Repeat** for the next label group.

A practical way to find multiple emails for selection: search for each email's
subject, select the checkbox, then modify the search query for the next email
without navigating away. Alternatively, if several emails share a common sender
or keyword, search for that to surface them together.

### One-at-a-time fallback

If batch selection proves unreliable (Gmail's UI can be unpredictable), fall
back to processing emails individually:

1. **Find the email.** Search by subject line (or sender if the subject is too
   generic). Use `find` to locate the search input, type the query, press Enter.
2. **Open the email.** Click the matching conversation. Use `find` and
   `read_page` to locate elements — don't hardcode coordinates, because Gmail's
   layout shifts depending on screen size, overlays, and rendering state.
3. **Apply the label.** Click the Labels button, search for the label, check it,
   and apply.
4. **Archive if Suspected Junk.** Click the Archive button after labeling.
5. **Confirm and navigate back.** Take a screenshot, then return to the inbox.

### Tips for Gmail automation

- **Use `find` for element refs, not hardcoded coordinates.** Gmail's layout
  shifts depending on screen size, zoom level, and rendering state. Coordinate-
  based clicking on label checkboxes is unreliable — the picker position changes
  slightly each time it opens. Instead:
  - Use `find("Action Needed checkbox")` to get the exact `menuitemcheckbox` ref
  - Use `find("Apply button")` to get the exact `menuitem` ref for Apply
  - Click using `ref` parameter instead of `coordinate` — this is significantly
    more reliable
- **Always verify after Apply.** Take a screenshot after clicking Apply to
  confirm the label badge appears next to the email subject. If it didn't take,
  re-open the picker and try again using refs.
- **Batch labeling from search results is fastest.** Select multiple email
  checkboxes in search results, then click the toolbar Labels icon. This applies
  the label to all selected emails in one operation. Gmail shows a confirmation
  toast like "5 conversations added to 'Informational'."
- **Use the exclusion query for verification.** After labeling, search for
  `in:inbox -label:Suspected-Junk -label:Action-Needed -label:Informational`
  to find any emails that were missed. This is the most reliable way to catch
  labeling failures.
- **Wait after navigation.** Use `wait` (1-2 seconds) after searching, opening
  emails, or clicking the Labels icon. Gmail's UI is asynchronous and elements
  may not appear immediately.
- The label picker has a search field — use it to find labels quickly rather
  than scrolling through a long list.
- If Gmail shows a loading overlay or interstitial, wait for it to clear before
  interacting with elements beneath it.

---

## Step 6 — Log results

Update (or create) `email-triage-log.md` in the user's working folder.

### Log format

The log is a Markdown table with these columns:

| Date | From | Subject | Label | Rule | Reason | Feedback |
|---|---|---|---|---|---|---|

- **Date**: The date of triage (today's date), not the email's sent date.
- **From**: Sender name (not full email address, to keep the table readable).
- **Subject**: Email subject line, truncated if very long.
- **Label**: The classification applied (Suspected Junk, Unknown, etc.).
- **Rule**: Which rule from triage-rules.md triggered the match (e.g.,
  "Rule 2 — Newsletters") or "No match" for Unknown.
- **Reason**: A brief explanation of why the rule matched (e.g., "Contains
  unsubscribe link, sent from noreply@example.com via Mailchimp").
- **Feedback**: Left empty — this column is for the user to annotate later
  (e.g., "false positive", "correct", "should be Action Needed").

### What to log

Log all emails classified as **Suspected Junk** or **Unknown**. These are the
decisions most likely to need user review.

You don't need to log Action Needed or Informational emails — those stay in the
inbox and the user will see them naturally.

### Append behavior

Add new entries at the **top** of the table (newest first), below the header
row. If the file doesn't exist yet, create it with the header row and then add
entries. If the file already has entries from previous sessions, preserve them
and add new rows above.

---

## Adapting the rules

After triage is complete, if the user points out mistakes (false positives or
false negatives), offer to update `triage-rules.md` to prevent the same error
in future sessions. Common updates include:

- Adding a sender or domain to the Trusted Domains list.
- Adding a newsletter to the Trusted Sources list.
- Adding a new exception to an existing rule.
- Creating a new rule pattern for a category of email the rules don't cover yet.

Always explain what change you'd make and why before editing the file. The user
should approve rule changes since they affect all future triage sessions.

---

## Reference files

| File | Purpose |
|---|---|
| `references/triage-rules-template.md` | Blank template the user copies and customizes with their own rules. Share this if `triage-rules.md` is missing from the working folder. |
