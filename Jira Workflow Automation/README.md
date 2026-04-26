<div align="center">

[![Built by Suhasini Ramesh](https://img.shields.io/badge/Built%20by-Suhasini%20Ramesh-blue?style=for-the-badge)](https://linkedin.com/in/suhasiniramesh)
[![GitHub Stars](https://img.shields.io/github/stars/rameshsuhasini/What-to-cook-app?style=for-the-badge&logo=github)](https://github.com/rameshsuhasini/What-to-cook-app)
[![n8n](https://img.shields.io/badge/Powered%20by-n8n-orange?style=for-the-badge)](https://n8n.io)
[![Claude AI](https://img.shields.io/badge/AI-Claude%20Haiku-purple?style=for-the-badge)](https://anthropic.com)

</div>

# GitHub → Jira Automation with Claude AI

An end-to-end workflow automation that eliminates manual Jira updates during development. Every GitHub action — pushing code, creating a branch, opening or merging a PR — automatically updates the right Jira ticket with an AI-generated comment and moves it to the correct status.

Built with **n8n**, **Jira Cloud API**, **GitHub Webhooks**, and **Anthropic Claude AI**.

---

## What It Does

| GitHub Event | Jira Comment | Status Change |
|---|---|---|
| Push commit (`DEV-25 fix login`) | ✅ AI summary added | 🔵 In Progress |
| Branch created (`feature/DEV-25-login`) | ✅ AI summary added | 🔵 In Progress |
| Pull Request opened | ✅ AI summary added + PR description auto-filled | 🟡 In Review |
| Pull Request merged | ✅ AI summary added | 🟢 Done |

**Additional features:**
- 📧 Daily Jira digest email every morning (HTML formatted, grouped by status)
- 🤖 Claude AI rewrites commit messages into professional Jira comments
- 🔁 Retry logic — Claude API retries 3 times before falling back to raw commit message
- 🚫 Duplicate prevention — won't spam Jira if multiple events fire within 10 minutes
- 🚨 Error handler — emails you if any part of the workflow fails
- 🔀 Merge commit detection — ignores merge commits so PR merged status isn't overwritten

---

## Architecture

```
GitHub Event (push / PR / branch)
        │
        ▼
GitHub Webhook
        │
        ▼
Route by Event (Switch Node)
   ┌────┴────────┬─────────────┐
   ▼             ▼             ▼
Extract       Extract       Extract
from Push     from PR       from Branch
   │             │               │
   │         Get Jira        (merges into
   │         Ticket Details   main flow)
   │             │
   │         Merge PR Data
   │             │
   │         Update PR Description (GitHub API)
   │             │
   └─────────────┴───────────────┘
                 │
                 ▼
           Has Jira ID?
                 │ yes
                 ▼
        Get Recent Comments
                 │
                 ▼
           Is Duplicate?
                 │ no
                 ▼
        Comment Summarize (Claude prompt)
                 │
                 ▼
        Claude API (claude-haiku-4-5)
        [Retry x3 → fallback to raw message]
                 │
                 ▼
        Merge Claude Response
                 │
                 ▼
        Add Jira Comment
                 │
                 ▼
        Pick Transition (push→In Progress,
                         PR open→In Review,
                         PR merge→Done)
                 │
                 ▼
        Update Jira Status
```

---

## Workflows Included

| File | Description |
|---|---|
| `Jira Automation.json` | Main workflow — GitHub → Jira sync |
| `Daily Digest.json` | Scheduled 8am email with all tickets by status |
| `Error Handler.json` | Email alert when any workflow node fails |

---

## Tech Stack

- **n8n** — workflow automation engine (self-hosted)
- **GitHub Webhooks** — push, pull_request, create events
- **Jira Cloud REST API v3** — comments, transitions, issue search
- **Anthropic Claude AI** (`claude-haiku-4-5-20251001`) — commit message summarisation
- **Gmail API** — daily digest + error alerts
- **Node.js / JavaScript** — custom logic in n8n Code nodes

---

## Setup Guide

### Prerequisites
- n8n instance (self-hosted or cloud)
- Jira Cloud account with API token
- GitHub repository
- Anthropic API key
- Gmail account

---

### Step 1 — Import Workflows

1. Open your n8n instance
2. Go to **Workflows → Import from file**
3. Import all 3 JSON files one by one
4. Save each workflow (do NOT activate yet)

---

### Step 2 — Set Up Credentials

#### Jira (Native n8n credential)
1. Go to **Credentials → New → Jira Software Cloud API**
2. Enter your Jira email and API token
3. Generate token at: https://id.atlassian.com/manage-profile/security/api-tokens

#### Anthropic Claude API
1. Go to **Credentials → New → HTTP Header Auth**
2. Name it: `Header Auth account`
3. Header Name: `x-api-key`
4. Header Value: your Anthropic API key from https://console.anthropic.com

#### GitHub Token (for PR description auto-fill)
1. Go to **Credentials → New → HTTP Header Auth**
2. Name it: `GitHub Token`
3. Header Name: `Authorization`
4. Header Value: `token ghp_YOURTOKEN`
5. Generate token at: https://github.com/settings/tokens (needs `repo` scope)

#### Gmail
1. Go to **Credentials → New → Gmail OAuth2**
2. Follow the OAuth flow to connect your Gmail account

---

### Step 3 — Configure Jira Transition IDs

Find your project's transition IDs by calling:
```
GET https://your-org.atlassian.net/rest/api/3/issue/DEV-1/transitions
```

Then open the **Pick Transition** node and update the transition map:
```javascript
const transitionMap = {
  push: 'YOUR_IN_PROGRESS_ID',
  branch_created: 'YOUR_IN_PROGRESS_ID',
  pr_opened: 'YOUR_IN_REVIEW_ID',
  pr_merged: 'YOUR_DONE_ID'
};
```

---

### Step 4 — Set Up GitHub Webhook

1. In n8n, open the **Github Webhook** node and copy the webhook URL
2. Go to your GitHub repo → **Settings → Webhooks → Add webhook**
3. Paste the URL into **Payload URL**
4. Set **Content type** to `application/json`
5. Select **Let me select individual events** and check:
   - ✅ Pushes
   - ✅ Pull requests
   - ✅ Branch or tag creation
6. Click **Add webhook**

---

### Step 5 — Configure Daily Digest

1. Open the **Daily Digest** workflow
2. Update the **Get Updated Tickets** node with your Jira project key
3. Update the **Gmail** node with your email address
4. Set the schedule time in the **Schedule Trigger** node (default: 8am)

---

### Step 6 — Activate Workflows

Activate in this order:
1. **Error Handler** first
2. **Jira Automation**
3. **Daily Digest**

---

### Step 7 — Test

Push a test commit with a Jira ticket ID in the message:
```bash
git commit -m "DEV-25 implement login page"
git push
```

Check your Jira ticket — you should see a new AI-generated comment and the status should have moved to In Progress.

---

## Commit Message Convention

The workflow detects Jira ticket IDs in the format `PROJECT-123`:

| ✅ Works | ❌ Won't work |
|---|---|
| `DEV-25 fix login bug` | `fix login bug` |
| `feat: [DEV-25] add auth` | `dev25 add auth` |
| Branch: `feature/DEV-25-login` | Branch: `feature/login` |

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Webhook not receiving events | Make sure n8n is publicly accessible. Use ngrok or Cloudflare Tunnel for local testing. |
| Jira comment not appearing | Check Jira credentials and that ticket ID format matches your project key |
| Status not updating | Verify transition IDs match your Jira workflow — IDs differ per project |
| Claude not responding | Verify Anthropic API key is correct. Workflow falls back to raw commit message automatically after 3 retries. |
| No Jira ID detected | Confirm commit message or branch name contains `PROJECT-123` format |
| Daily digest not sending | Check Gmail OAuth credential is still valid |

---

## Show Your Support

If this project helped you or saved you time, please consider:
- ⭐ **Starring this repo** — it helps others find it
- 🔗 **Connecting on [LinkedIn](https://linkedin.com/in/suhasiniramesh)** — I'd love to hear how you're using it
- 🐛 **Opening an issue** if you run into problems — I'll help you fix it

---

## Author

Built by **Suhasini Ramesh**  
[LinkedIn](https://linkedin.com/in/suhasiniramesh) · [GitHub](https://github.com/rameshsuhasini)
