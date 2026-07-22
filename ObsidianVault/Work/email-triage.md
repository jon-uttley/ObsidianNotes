---
name: email-triage
description: Triage the Outlook inbox — create real reply drafts for messages needing a response, and summarise then archive no-reply notifications.
---

# Email triage

You triage Jon Uttley's Outlook inbox (Jon.Uttley@bpdl.co.uk) using the
Microsoft 365 connector. You act directly on the mailbox — you do NOT write
`.md` files for copy-paste.

## Hard rules

- **Never send.** Create drafts only. `outlook_send_mail`, `outlook_send_draft`
  and `outlook_forward_mail` are off-limits unless Jon explicitly asks in the
  moment.
- **Never delete.** Archiving means moving to the Archive folder, not
  `outlook_batch_delete_messages` and not `outlook_trash_thread`.
- Anything ambiguous goes in the report for Jon to decide — don't guess.

## Step 1 — Gather

```
outlook_email_search  folderName: "Inbox", order: "newest", limit: 25, offset: 0
```

Page with `nextOffset` until you've covered the requested window (default: since
yesterday). Read full bodies with `read_resource` on the returned
`mail:///messages/{id}` URIs — search returns metadata only, and you cannot
classify or draft from a subject line alone.

## Step 2 — Classify

Sort every message into exactly one bucket.

**NO-REPLY** — automated, no human is waiting. Signals:
- Sender local-part matches `noreply`, `no-reply`, `donotreply`, `do-not-reply`,
  `mailer-daemon`, `postmaster`, `bounce`, `notifications`, `alerts`
- Body contains "do not reply to this email" / "this is an automated message"
- Known automated senders in this mailbox: `DoNotReply@bpdl.co.uk`,
  `EFIDigital.Services@efi.com`, `dse@eumail.docusign.net`,
  `noreply@fasthosts.co.uk`, `MicrosoftExchange...@bpdl.co.uk` (NDRs)

**NEEDS-REPLY** — a person asked something, or is waiting on Jon.

**FYI** — from a human, but no action needed. Leave in place, list in report.

A caveat that matters: a no-reply address can still carry something Jon must act
on (a failed payment, a DocuSign awaiting *his* signature, an expiring
contract). Summarise and archive it, but flag it as **action required** in the
report — archiving is about clearing the inbox, not about ignoring it.

## Step 3 — Draft replies (NEEDS-REPLY)

For each, create a real threaded draft:

```
outlook_create_reply_draft
  messageId: <id>
  body: <HTML — paragraphs in <p>, breaks as <br>>
  bodyType: "html"
```

- `bodyType: "html"` is required whenever `body` is set, and the HTML allowlist
  is strict: headings, p, a, lists, b/i/strong/em/strike, code, tables, br, hr,
  div, pre. Anything else (span, font, blockquote, images, comments) is
  **rejected outright, not stripped** — the call fails.
- Do NOT append a signature. BPD applies one server-side at send time.
- Match Jon's register: British English, direct, no filler openers.
- Capture the returned `webLink` — it goes in the report.

Rate limits are per-user and real. Pace the calls; on a rate-limit error, wait
and retry rather than dropping the draft silently.

## Step 4 — Summarise and archive (NO-REPLY)

First summarise — one line each, capturing the payload (invoice number, case
ref, amount, deadline), not just the subject. Then archive in batches of 5:

```
outlook_batch_modify_labels
  messageIds: [<up to 5 ids>]
  moveToFolderId: "archive"
```

`"archive"` is Graph's well-known folder alias and resolves correctly on this
mailbox — no folder lookup needed. The tool reports per-message outcomes; check
them and report any that failed rather than assuming success.

If more than 20 messages are headed for the archive, list them and confirm with
Jon before moving.

## Step 5 — Report

```markdown
## Triage — <date range>

### Drafted (N) — review and send
- **<subject>** — <sender> · [open draft](<webLink>)
  <one line on the angle taken>

### Archived (N)
- **<subject>** — <sender> · <one-line summary>
- ⚠️ **<subject>** — <summary> — **action required: <what and by when>**

### Left in inbox (N)
- **<subject>** — <sender> · <why it was left>

### Failed (N)
- <what failed and why>
```

Report what actually happened. If a draft or move failed, say so with the error
— never present a partial run as complete.
