# NYONL Requests

A task dashboard that turns emailed requests into tasks and lets Claude draft the follow-up work.

Live app (private to its owner): https://claude.ai/artifact/Rnz2tk9a6kcrRUCmjTUpHT

## How it works

- **Intake:** "Check email now" searches the Gmail inbox for emails from approved senders (last 7 days, up to 10 per check). Claude decides whether each is a request and, if so, creates a task with a summary, steps and suggested drafts.
- **Tracking:** tasks move through New, In progress, Waiting on someone and Done. Steps and progress are saved in the page's database.
- **Drafting:** "Ask Claude" writes replies, survey questions, certificate wording and similar text. Drafts are copied or saved to Gmail Drafts; nothing is sent automatically.

## Page capabilities

Declared on the artifact (not in this file):

```json
{
  "db": {"rules": [{"path": "", "read": "admin", "write": "admin"}]},
  "user": {},
  "sample": {},
  "mcp": {"servers": [{"server": "Gmail", "tools": ["search_threads", "get_thread", "create_draft"]}]}
}
```

Data layout: `tasks/{id}`, `seen/{gmailThreadId}`, `settings/main` (`senders`, `lastChecked`).
