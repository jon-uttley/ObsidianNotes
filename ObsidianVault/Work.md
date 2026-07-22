The three changes that matter:

1. Real drafts — outlook_create_reply_draft with the message id gives you a properly threaded draft in Outlook with the quoted original and correct reference headers. Far better than a .md you paste, because the threading survives.
2. Archive via well-known alias — outlook_batch_modify_labels with moveToFolderId: "archive". Note the batch cap is 5 message ids per call, not 50 (the 50 cap is on batch_delete). Easy thing to get wrong.
3. No-reply ≠ ignorable — I added a rule to flag action-required items even while archiving them. Looking at your Archive folder, that bucket includes DocuSign completions, EFI support cases, and Fasthosts invoices — you don't want a "please sign" quietly filed away.

Two gotchas baked in:
- The HTML allowlist rejects rather than sanitises. A <span> or <blockquote> fails the whole call — a plausible cause of silent breakage if the agent writes rich HTML.
- No signature in the body. Worth confirming: does BPD apply signatures server-side via Exchange transport rules? If not, the drafts will go out bare and that line needs changing.