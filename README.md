# Vidya OS

Vidya is an AI data-analysis and research agent. Upload a dataset or ask a question, and Vidya plans the approach, writes and runs the code itself, fixes its own mistakes, and hands you back a written answer with charts — no notebook, no copy-pasting, no debugging on your end.

🔗 **Try it now:** [navi-os.cc](https://navi-os.cc) — no account needed to start.

## Getting started

Open the chat and start typing — a temporary account is provisioned for you automatically in the background, with no sign-up form. It comes with a smaller daily quota than a full account, and is cleaned up automatically after it's been inactive for a while.

Create a real (still free) account when you want:
- Your chat history and memory to persist between visits
- A higher daily query limit
- Subagents and external service connections

## What you can do

### Analyze a dataset

1. Upload a CSV or Excel file.
2. Ask your question in plain English — e.g. *"which product category grew the fastest this quarter?"*
3. Vidya writes and runs the analysis and gives you a written answer plus any relevant charts.

If the first attempt fails, Vidya doesn't just hand the error back to you. It diagnoses what went wrong, researches a fix if needed, rewrites the code, and tries again automatically — only asking you for help once it's genuinely stuck, and telling you exactly what it needs.

### Ask a research question

You don't need a dataset to ask Vidya something. If your question needs current, real-world information — a price, a ranking, anything that could have a different answer tomorrow than today — Vidya searches the web and cites its sources. Ask a timeless question instead and it just answers directly, no search needed.

### Build a knowledge base

Upload documents — PDF, Word, Markdown, plain text, JSON, CSV — to build a private knowledge base, then ask questions about them in the same chat. Vidya searches across everything you've uploaded and answers with source attribution.

### Create a subagent

Subagents are specialized personas you create for a particular domain — a "Finance Analyst," a "Support Triage Agent," whatever fits your use case. Describe what you want and Vidya generates its name, icon, and persona for you. Each subagent keeps its own conversation memory, separate from your main chat and from every other subagent.

A subagent can also be connected to your own AWS S3 bucket with a one-click setup (a pre-filled CloudFormation stack you launch in your own AWS account) so it can read and write files there directly — Vidya never holds standing access to your bucket, only a scoped role for that one subagent.

### Connect your tools

Connect Vidya to Slack, Notion, Jira, Asana, Stripe, Canva, Zapier, or any other service that supports the Model Context Protocol — not just a fixed list, any compliant server. Once connected, just ask in plain language: *"search #project-alpha for the latest update and write a summary page in Notion."*

Anything that only reads information (a search, a fetch) runs immediately. Anything that changes something outside Vidya — sending a message, creating a page, updating a record — is held for your explicit approval first. You'll see exactly what it's about to do before it happens, and you can approve it, reject it, or opt a specific tool into running automatically once you trust it.

## Plans

| | Free | Pro — $15/mo |
|---|---|---|
| Chat queries | 10/day | 200/mo included, then $0.30/query (capped at 250) |
| Subagents | Included | Unlimited |
| AWS S3 integration | ✅ | ✅ |
| Persistent memory | ✅ | ✅ |
| Priority access | ✅ | ✅ |

## Privacy & your data

- An uploaded dataset or document is held in memory (or a short-lived cache) for your session — it's never written to a database.
- Chat message text is saved so your history persists, with personal information redacted before anything is stored in long-term memory.
- Generated charts and computed results are available to download from the chat itself, but aren't kept beyond that.
- A write-capable action on a connected tool never runs without your approval unless you've explicitly opted it into running automatically.
