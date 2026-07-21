# Morning Call Scoring — Setup Guide

This is a **template** skill. It automates: pull yesterday's first-touch sales
calls from a call-recording tool, score each against a discovery/BANT rubric,
log a row to a database, update your CRM, and post a summary to chat.

It was genericized from a working install, so every org-specific value has been
replaced with a `<PLACEHOLDER>`. **The skill will not run until you replace
them.** Nothing here works out of the box — budget ~30–60 min of setup.

---

## 1. Install the skill

Copy the whole `score-morning-call/` folder into either:

- `~/.claude/skills/score-morning-call/` — available in all your projects, or
- `<your-project>/.claude/skills/score-morning-call/` — shared with a repo/team.

Then it's invocable as `/score-morning-call`.

## 2. Connect the MCP servers

This skill assumes four connectors are authorized in your Claude Code
environment. Each has an instance-specific **server ID** that appears in tool
names as `mcp__<SERVER_ID>__<tool>`. Yours will differ from the placeholders.

| Role in the skill | Placeholder | What you need |
|---|---|---|
| Call recorder / transcripts | `<CALL_RECORDER_MCP_ID>` | Grain, Gong, Fireflies, etc. — must expose list/fetch meeting + transcript tools |
| CRM | `<CRM_MCP_ID>` | HubSpot (as written), or adapt for Salesforce, etc. |
| Database / logging | `<DATABASE_MCP_ID>` | Notion (as written), or adapt for Airtable, etc. |
| Chat / notifications | Slack bot token (see §5) | Slack (posts via bot token + curl, not an MCP) |

Find your server IDs: run `/mcp` in an interactive Claude Code session, or check
your connector settings. Then do a find-and-replace across `SKILL.md`.

## 3. Fill in `summary-target.txt`

Open `summary-target.txt` and replace every `<PLACEHOLDER>` with your real
channel ID, database URLs, etc. The skill reads this file at runtime, so getting
it right here saves editing SKILL.md.

## 4. Customize the scoring rubric

`scoring-rubric.txt` is the scoring brain. The framework (BANT, discovery
categories, objection handling, analysis requirements, output format) is
reusable as-is. **But the `COMPANY CONTEXT`, `PRODUCTS`, and `PRIMARY USE CASES`
sections are stubbed** — the scorer can't judge fit without knowing what you
sell. Paste in your own product catalog and ICP before first run.

## 5. Slack bot (if using Slack)

The skill posts as a dedicated bot so consecutive audits don't get visually
grouped by Slack. To replicate:

1. Create a Slack app with `chat:write` scope; install it to your workspace.
2. Invite the bot to your destination channel.
3. Put its bot token in `.claude/settings.local.json` env as
   `<SLACK_BOT_TOKEN_ENV>` (keep it out of git).
4. Set the channel ID and app ID in `summary-target.txt`.

If you'd rather post via a Slack MCP instead, swap the `curl` blocks in Step 8
for your Slack MCP's send-message tool (note the grouping tradeoff called out
there).

## 6. Placeholder reference

Every token to replace, and what it was in the source install's role:

| Placeholder | Meaning |
|---|---|
| `<YOUR_COMPANY>` | Your company name (the vendor being sold) |
| `<HOST_1>`, `<HOST_2>` | The AE(s) whose calls get scored |
| `<INTERNAL_EMAIL_DOMAINS>` | Your internal domains, used to detect external attendees |
| `<TIMEZONE>` | IANA tz for the "prior business day" window, e.g. `America/Chicago` |
| `<CALL_RECORDER_MCP_ID>` | Server ID of your call-recording connector |
| `<CRM_MCP_ID>` | Server ID of your CRM connector |
| `<DATABASE_MCP_ID>` | Server ID of your database connector |
| `<CRM_OWNER_NAME>` / `<CRM_OWNER_EMAIL>` | The account the CRM connector authenticates as |
| `<CRM_OWNER_ID>` / `<CRM_ACCOUNT_ID>` | CRM internal IDs for that account |
| `<MARKETING_PIPELINE_ID>` | CRM pipeline used for un-qualified inbound (MQL) deals |
| `<MARKETING_FIRST_MEETING_STAGE_ID>` | Entry stage in that pipeline |
| `<MARKETING_DISQUALIFIED_STAGE_ID>` | "Disqualified" stage in that pipeline |
| `<PROSPECT_PIPELINE_ID>` / `<PROSPECT_QUALIFICATION_STAGE_ID>` | Target pipeline+stage a human promotes SQLs into |
| `<SLACK_CHANNEL_ID>` / `<SLACK_CHANNEL_NAME>` | Destination channel |
| `<SLACK_BOT_APP_ID>` | Your Slack app's ID |
| `<SLACK_BOT_TOKEN_ENV>` | Env var name holding the bot token |
| `<NOTION_DATA_SOURCE_URL>` / `<NOTION_DB_URL>` / `<NOTION_PARENT_PAGE>` | Your database identifiers |

## 7. Notes on what was stripped

- **Incident/tuning logs** in the original (specific dates, prospect names, and
  channel-drift fixes) were replaced with generic explanations of *why* each
  guardrail exists. As you run this, re-add your own dated incident notes — they
  are the most valuable part of the skill over time.
- **CRM auto-push is pre-authorized and unattended** in this skill. Review Step 6
  carefully and decide whether you want that, or whether you want a human
  approval gate. The `--dry-run` manual mode is the safe way to start.
- The database schema (column names/types) in Step 7 mirrors the source Notion
  DB. Recreate an equivalent database and adjust the property list to match.
