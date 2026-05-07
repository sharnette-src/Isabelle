# Isabelle

Isabelle is Propel's Talent Acquisition Operations AI assistant, built with Node.js, Slack Bolt, and Claude (Anthropic). She thinks like a 15-year TA Ops veteran — part of the team, not above it.

---

## What Isabelle Does

| Flow | Trigger | What happens |
|---|---|---|
| **Scorecard handoff** | Interviewer submits scorecard in Ashby | Isabelle DMs the next interviewer with a signal summary and one follow-up question to probe |
| **Interviewer coaching** | Same as above | If Claude detects missed signals, Isabelle DMs the interviewer with developmental feedback |
| **Overdue reminders** | Scheduled check every 2 hours | Isabelle sends warm nudges for scorecards 24h–7 days overdue |
| **Escalation** | Same scheduled check | Scorecards 7+ days overdue get escalated to the recruiter |
| **Debrief prep** | All scorecards submitted for a role | Isabelle posts a candidate synthesis and two discussion questions to the hiring channel |

---

## Tech Stack

- **Node.js** + Slack Bolt (`@slack/bolt`)
- **Socket Mode** — no public URL required; Isabelle connects outbound to Slack
- **Anthropic SDK** (`@anthropic-ai/sdk`) — Claude powers all messaging
- **Express** — receives Ashby webhook events
- **node-cron** — runs overdue/escalation checks every 2 hours

---

## Project Structure

```
isabelle/
├── index.js              # Main file: Slack app, webhook receiver, scheduler
├── handlers/
│   ├── scorecard.js      # Scorecard handoff + channel summary
│   ├── overdue.js        # Overdue reminders
│   ├── escalation.js     # Escalation flags
│   ├── compare.js        # Debrief prep / candidate comparison
│   └── coaching.js       # Interviewer coaching
├── utils/
│   ├── claude.js         # Anthropic API wrapper + Isabelle's personality
│   └── slack.js          # Slack message builders + helpers
├── test/
│   └── simulate.js       # Test script with mock Ashby payloads
├── .env.example
└── package.json
```

---

## Setup

### 1. Prerequisites

- Node.js v18+
- Slack workspace with admin access
- Anthropic API key ([console.anthropic.com](https://console.anthropic.com))
- Ashby account with webhook access

### 2. Install dependencies

```bash
cd isabelle
npm install
```

### 3. Create and configure the Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**
2. Name it **Isabelle**, select your workspace

**OAuth & Permissions** — add these Bot Token Scopes:
- `chat:write`
- `users:read`
- `users:read.email`
- `app_mentions:read`
- `commands`

3. Install the app to your workspace and copy the **Bot User OAuth Token** (`xoxb-...`)
4. Under **Basic Information**, copy the **Signing Secret**

**Socket Mode** — enable it under the Socket Mode tab, generate an App-Level Token with the `connections:write` scope, copy the token (`xapp-...`)

### 4. Configure environment variables

```bash
cp .env.example .env
```

Fill in all values in `.env`:

```bash
SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
SLACK_APP_TOKEN=xapp-...
ANTHROPIC_API_KEY=sk-ant-...
ASHBY_API_KEY=your-ashby-api-key-here
PORT=3000
NODE_ENV=development
```

### 5. Run Isabelle

```bash
npm start
```

You should see:

```
⚡️ Isabelle (Slack Bolt) is running!
🌐 Webhook server is running on port 3001
   Health check: http://localhost:3001/health
   Webhooks: http://localhost:3001/webhooks/ashby
[INFO] socket-mode:SocketModeClient:0 Now connected to Slack
✨ Isabelle is ready to help with Talent Acquisition Operations!
```

### 6. Invite Isabelle to your hiring channels

In each Slack channel Isabelle should post to:
```
/invite @Isabelle
```

---

## Ashby Webhook Configuration

Point Ashby webhooks to:
```
http://your-server:3001/webhooks/ashby
```

Isabelle listens for these events:

| Ashby event | Flow triggered |
|---|---|
| `scorecard.submit` | Scorecard handoff + coaching |
| `application.stage.move` (with `all_scorecards_completed: true`) | Debrief prep |

> **Note:** Verify the exact event names in your Ashby account under Settings → Webhooks. Update the switch cases in `index.js` if they differ.

### Expected payload shape

```json
{
  "action": "scorecard.submit",
  "payload": {
    "candidate_name": "Jane Doe",
    "job_title": "Senior Engineer",
    "interviewer_email": "alex@joinpropel.com",
    "interviewer_name": "Alex Chen",
    "next_interviewer_email": "jordan@joinpropel.com",
    "scorecard_url": "https://app.ashbyhq.com/scorecards/12345",
    "hiring_channel_id": "C01234567",
    "scorecard_content": "...",
    "interview_stage": "Hiring Manager Interview",
    "interview_focus_area": "Product Strategy"
  }
}
```

> You may need to map Ashby's native payload format to this shape using a middleware transform or Ashby's custom field configuration.

---

## Testing

Update the constants at the top of `test/simulate.js` with real values from your workspace:

```js
const HIRING_CHANNEL_ID = 'C...';           // Slack channel ID (right-click channel → View details)
const INTERVIEWER_EMAIL = 'you@joinpropel.com';
const NEXT_INTERVIEWER_EMAIL = 'teammate@joinpropel.com';
```

Then with Isabelle running in one terminal, run in a second terminal:

```bash
npm test
```

This tests: health check, scorecard handoff + coaching, debrief comparison, and overdue reminder.

---

## Overdue Reminders (Production)

The scheduled job runs every 2 hours but the `getOverdueScorecards()` function in `index.js` currently returns an empty array. For production, replace it with a real Ashby API call that returns pending scorecards and calculates hours overdue.

Each returned object should match the shape in `handlers/overdue.js`.

---

## Customization

**Isabelle's personality** — edit `ISABELLE_SYSTEM_PROMPT` in `utils/claude.js`

**Message length** — each handler passes a `maxTokens` value to `callClaude()`. Increase it for longer responses.

**Add a new flow** — create a handler in `handlers/`, import it in `index.js`, and wire it to a webhook action or slash command.

---

## Deployment

Set all five environment variables in your production environment (same as `.env`, no file needed). Then run:

```bash
NODE_ENV=production npm start
```

Recommended platforms: Railway, Render, Heroku, AWS ECS. Use PM2 or your platform's process manager to keep Isabelle running.

> The test endpoints (`/test/overdue`) are disabled automatically when `NODE_ENV=production`.

---

## Troubleshooting

**Messages not appearing in channel** — make sure Isabelle has been invited (`/invite @Isabelle`)

**DMs not sending** — confirm the email addresses in the webhook payload match Slack account emails exactly

**Socket Mode not connecting** — verify `SLACK_APP_TOKEN` starts with `xapp-` and has the `connections:write` scope

**Claude API errors** — check your Anthropic API key and account quota at console.anthropic.com

---

## Transferring Ownership

1. Push the repo to GitHub (private)
2. Rotate all credentials before sharing — Slack tokens at api.slack.com/apps, Anthropic key at console.anthropic.com
3. Transfer the Slack app under App Management → Transfer Ownership
4. New owner fills in their own `.env` values and runs `npm start`

---

**Built by the Propel team** · Isabelle makes hiring a better team sport.
