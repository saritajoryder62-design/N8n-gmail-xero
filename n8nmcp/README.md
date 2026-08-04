# N8n Gmail → Xero Auto-Invoice

Watches a Gmail inbox for incoming bills/invoices/receipts and automatically creates a matching draft bill in Xero, with the original attachment uploaded onto it.

## Status

Working end-to-end for a single Xero organisation. Currently a personal automation — the goal is to turn it into a product other people can sign up for by connecting their own Gmail and Xero accounts. See [`Xero-Auto-Invoice-Platform-Build-Plan.pdf`](./Xero-Auto-Invoice-Platform-Build-Plan.pdf) for the full phased plan (OAuth app setup → sign-up UI → per-user provisioning → launch).

## How the workflow works

Built and run in [n8n](https://n8n.io/). Flow:

1. **Gmail Trigger** — polls for new mail matching `has:attachment (invoice OR bill OR receipt)`, with attachments downloaded.
2. **Parse Email Headers** — extracts sender name, sender email, and subject from the parsed email.
3. **Has Attachment** — filters out anything without a binary attachment.
4. **Xero - Find Contact by Email** — looks up the sender as an existing Xero contact.
5. **Contact Found?** — branches:
   - found → straight to invoice creation
   - not found → **Xero - Create Contact** first, using the sender's name/email
6. **Xero - Create Draft Invoice** — creates a draft `ACCPAY` bill referencing the email subject.
7. **Reattach Attachment** → **Xero - Upload Attachment** — re-attaches the original binary and uploads it onto the created bill via the Xero API.

A snapshot of this workflow's nodes/connections is kept at [`workflows/xero-flow.json`](./workflows/xero-flow.json) for reference and version history — see [Workflow snapshots](#workflow-snapshots) below for how it relates to the live version in n8n.

## Repo structure

```
n8nmcp/
├── README.md                                  # this file
├── Xero-Auto-Invoice-Platform-Build-Plan.pdf  # phased plan to scale this into a multi-user product
├── workflows/
│   └── xero-flow.json                         # portable snapshot of the n8n workflow (credentials stripped)
├── .mcp.json / mcp.json                        # local-only MCP server config (gitignored, see below)
└── .claude/                                    # local Claude Code settings (gitignored)
```

## Workflow snapshots

`workflows/xero-flow.json` is a **manual, point-in-time export** — not a live sync. Credential IDs and the Xero tenant ID have been stripped/replaced with `YOUR_XERO_TENANT_ID` so it's safe to commit and reusable as a starting point on any n8n instance. It won't auto-update when the live workflow changes; re-export it when you want to snapshot a new milestone.

For always-in-sync version control of the live workflow, n8n has a built-in **Source Control** feature (Settings → Source Control in the n8n UI) that syncs directly to a git repo — worth adopting once there's a stable template worth protecting, but it's typically an Enterprise-licensed feature, so check availability on your instance first.

## Local setup

This project connects to n8n via the [n8n-mcp](https://www.npmjs.com/package/n8n-mcp) MCP server, configured through `.mcp.json` (used by Claude Code / MCP-compatible tools). That file contains a live n8n API key, so it is **gitignored** and never committed. To work on this project locally, create your own `.mcp.json` in the project root:

```json
{
  "mcpServers": {
    "n8n": {
      "command": "npx",
      "args": ["-y", "n8n-mcp"],
      "env": {
        "MCP_MODE": "stdio",
        "LOG_LEVEL": "error",
        "DISABLE_CONSOLE_OUTPUT": "true",
        "N8N_API_URL": "https://your-n8n-instance.example.com/",
        "N8N_API_KEY": "your-n8n-api-key"
      }
    }
  }
}
```

## Roadmap

See the build plan PDF for full detail. Short version:

1. ~~Working single-tenant automation~~ ✅
2. Register Google + Xero OAuth apps (one-time, developer-side)
3. Build a sign-up UI (Next.js + Tailwind) with "Connect Google" / "Connect Xero"
4. Provisioning backend: clone this workflow per user, wired to their own credentials
5. Test with a second real account end-to-end
6. Production readiness (OAuth app review, monitoring, privacy policy)
7. Launch

## Security note

This repo's root is the owner's Desktop folder rather than a dedicated project folder — only files under `n8nmcp/` are intentionally tracked. Never commit `.mcp.json`, `mcp.json`, `.env`, or anything containing API keys/OAuth secrets; they're covered by `.gitignore`.
