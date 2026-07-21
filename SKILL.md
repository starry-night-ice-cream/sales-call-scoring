---
name: score-morning-call
description: Pull every prior-business-day discovery/intro call hosted by <HOST_1> or <HOST_2> from the call recorder, score each with the <YOUR_COMPANY> Objective Revenue Auditor framework, append a row per call to the Notion Morning Call Audits database, attempt CRM note + deal-field updates per call, and post one Slack message per call to <SLACK_CHANNEL_NAME>. Designed for unattended morning automation but also invocable manually.
---

# Morning Call Scoring Routine

> **TEMPLATE NOTICE — read `SETUP.md` first.** Every `<PLACEHOLDER>` below must
> be replaced with your org's real values before this skill will run. This
> includes MCP server IDs (`mcp__<...>__tool` names), hosts, CRM pipeline/stage
> IDs, Slack channel + bot, and Notion database IDs. Nothing here works out of
> the box.

You are running unattended on behalf of the <YOUR_COMPANY> revenue team. There
is no human in the loop until after the run completes — the pre-authorization
for this routine is established during setup (see Step 6).

## Inputs

- **Scoring rubric:** `scoring-rubric.txt` in this skill folder — this is the scoring rubric and output format. Read it before scoring.
- **Target hosts:** <HOST_1>, <HOST_2>.
- **Target call type:** First discovery / intro call only. Skip follow-ups, demos, kickoffs, QBRs, renewals, internal calls, 1:1s.
- **Date window:** Prior business day in <TIMEZONE> (Mon–Fri only — on Mondays, look back to Friday).

## Step 1 — Resolve the date window

Compute the prior business day in `<TIMEZONE>`:
- If today is Mon, prior business day = last Fri.
- Else, prior business day = today − 1 day.

Convert to UTC bounds for call-recorder queries: `[YYYY-MM-DDT00:00:00<UTC_OFFSET>, YYYY-MM-DDT23:59:59<UTC_OFFSET>]` (mind daylight-saving offset changes).

## Step 2 — Pull candidate meetings from the call recorder

Use `mcp__<CALL_RECORDER_MCP_ID>__list_meetings` with the date range. For each meeting, also resolve the host via `list_workspace_users` if not already on the meeting record.

Filter to meetings where:
- Host email/name matches **<HOST_1>** or **<HOST_2>** (case-insensitive).
- At least one external participant (not on your `<INTERNAL_EMAIL_DOMAINS>`) is present.
- Meeting title does NOT contain any of: `follow up`, `follow-up`, `kickoff`, `kick off`, `QBR`, `renewal`, `expansion`, `internal`, `sync`, `1:1`, `1-1`, `team`, `pipeline review`, `forecast`, `standup`.
- **"demo" handling — DO NOT use as a blanket exclusion.** Prospect-side schedulers frequently put "demo" in first-meeting titles when they mean "first vendor meeting" — a genuine discovery call by someone in research mode. For any candidate meeting whose title contains `demo`, run a **call-history disambiguation check**:
  1. Resolve the prospect's company via `search_companies` using the external participant's email domain or the meeting title.
  2. Call `list_meetings` with that company filter AND a `before_datetime` set to this meeting's `start_datetime`.
  3. If the result is **empty** (no prior meetings with that company) → **include** the meeting as a discovery call.
  4. If the result is **non-empty** → exclude (this is a true product demo for an existing prospect).
  5. Log the disambiguation outcome ("first contact with {company} — include" / "prior meeting found — exclude") in the run output.
- Title positively suggests first call: contains any of `intro`, `discovery`, `disco`, `<> + <YOUR_COMPANY>`, `<> x <YOUR_COMPANY>`, `<YOUR_COMPANY> +`, `<YOUR_COMPANY> x`, or no follow-up signal AND it's the earliest call with that company in the recorder.

### Why the "demo" rule exists

A blanket "demo" exclusion will drop real first-touch discovery calls whenever
the prospect names the meeting "{Vendor} Demo for {Company}." The
disambiguation check above (first-contact = include) is what prevents that.
**Keep a dated log here of any call the filter wrongly missed**, so future
tuning has a record. When a run misses a call, use `--meeting-id <recorder-id>`
to backfill rather than re-running the whole date window.

Sort the surviving candidates by start time ascending. **Every surviving candidate is a target call** — the run scores all of them, not just the first.

If zero candidates survive: skip the run, post "No qualifying calls for {date}" to the Slack channel, exit cleanly. Do not push anything to the CRM.

## Step 3 — Fetch the full call payload

Steps 3 through 7 run **once per target call**. Process calls in start-time order. Failures on one call must not block the others — capture the failure and continue.

For each meeting, fetch in parallel:
- `fetch_meeting` (metadata + participants + linked CRM deal if the recorder has it)
- `fetch_meeting_transcript`
- `fetch_meeting_notes`
- `fetch_meeting_action_items`

## Step 4 — Resolve the CRM deal

Preference order:
1. If the meeting payload includes a linked CRM deal, use that `dealId`. Verify with `get_crm_objects(objectType="deals", objectIds=[id])`.
2. Else, find the prospect contact: take each external participant email, run `search_crm_objects(objectType="contacts", filters=[{propertyName:"email", operator:"EQ", value:<email>}])`. For the matched contact, search for associated open deals: `search_crm_objects(objectType="deals", filterGroups=[{associatedWith:[{objectType:"contacts", operator:"EQUAL", objectIdValues:[<contactId>]}]}], properties:["dealname","dealstage","amount","closedate","pipeline","description","hubspot_owner_id"])`.
3. If multiple deals: prefer the one whose `dealname` includes the company name from the call AND is in an open pipeline stage. If still ambiguous, pick the most recently updated open deal.
4. If still nothing: search by company. `search_companies` (recorder) or `search_crm_objects(objectType="companies", query=<companyDomainOrName>)`, then walk to associated deals.

Capture: `dealId`, current `dealname`, `dealstage`, `amount`, `closedate`, `description`, `hubspot_owner_id`, deal URL.

### No deal resolved → create one in the inbound/Marketing pipeline

If no deal can be resolved with reasonable confidence, **create a new deal in the Marketing pipeline** rather than skipping. This is the one and only automatic deal creation; it is a *creation*, not a *move* (see Step 6 — existing deals are never moved automatically).

Use `manage_crm_objects` with a `createRequest` (include `confirmationStatus: "CONFIRMATION_WAIVED_FOR_SESSION"`). Populate from the call:
- `dealname` = `{Company} <> <YOUR_COMPANY>`.
- `pipeline` = `<MARKETING_PIPELINE_ID>`.
- `dealstage` = `<MARKETING_FIRST_MEETING_STAGE_ID>` (the call already happened, so this is the correct entry stage).
- `amount` = the figure the call surfaced, if any (omit otherwise).
- `closedate` = only if the call surfaced an explicit timeline (ISO date string); omit otherwise.
- `hubspot_owner_id` = the host's owner ID (<HOST_1> or <HOST_2> — resolved in Step 6's owner lookup), not the connector's own account.
- Associate the prospect `contact` and `company` resolved above.

Capture the returned `dealId` and treat it as the resolved deal for Steps 5–8. In the run output and Slack summary, note "Created new Marketing deal (no prior deal found)".

If deal creation fails (permission or validation error): fall back to producing the audit + a strict JSON payload + the prospect contact ID and call URL in the Slack summary for manual entry. Do not retry.

## Step 5 — Score the call

Read `scoring-rubric.txt`. Apply the **ANALYSIS REQUIREMENTS** (1–11) using the transcript + notes + action items. Hold yourself to the calibration rules:
- Early stages cap at 40% close probability without strong BANT.
- Late stages need confirmed authority + timeline to exceed 70%.
- Operator voice only — no marketing language, no hedging.

Produce the **copyable text recap** exactly as specified in the OUTPUT FORMAT section (the fenced block format). This is the canonical artifact for the CRM note. **Skip the visualizer widget** — there is no UI in the automated run.

Also produce the structured field updates:
- `dealStageRecommended` (movement: Forward / Stay / Backward / Disqualify → mapped to actual pipeline stage IDs via `get_properties(objectType="deals", propertyName="dealstage")`)
- `amountRecommended` (the new $ figure if changed)
- `closeDateRecommended` (only if the call surfaced explicit timeline change; else leave unchanged)
- `descriptionRecommended` (a tightened ≤500-char operator summary appended to the existing description with a dated header: `\n\n--- Auto-audit {YYYY-MM-DD} ---\n{summary}`)

## Step 5b — MQL → SQL Disposition (Marketing pipeline only)

**Fires only when** the resolved deal's `pipeline` equals `<MARKETING_PIPELINE_ID>`. For deals in any other pipeline, **skip this step entirely** — emit no MQL→SQL value to any surface (CRM note, Notion column, Slack line). Hide, do not show "N/A".

### The 4 SQL gates

Score each gate as Pass / Fail using transcript + notes evidence:

1. **Authority confirmed** — A buyer or VP-level decision-maker is on the call (or named with a confirmed intro plan). A researcher / IC / "I'm scoping this for my team" voice is a Fail.
2. **Timeline ≤ 12 months** — An explicit pilot or evaluation window stated in writing or on the call. A vague "someday" or 18+ month build/pivot horizon is a Fail.
3. **Active or quantified pain** — A specific loss, threshold, regulatory pressure, or business problem <YOUR_COMPANY> directly addresses. Curiosity / educational interest / "future capability" framing is a Fail.
4. **No structural blocker** — Required infrastructure (data feeds, integrations, etc.) is live OR has a committed go-live date. "In pockets," "in roadmap," or "in discovery" is a Fail.

### Decision rule

| Gates passed | Disposition |
|---|---|
| 4 of 4, no fatal flaw | `Convert MQL→SQL` |
| 3 of 4, no fatal flaw | `Convert MQL→SQL` (flag the missing gate as a risk in the rationale) |
| ≤ 2 of 4 | `Hold in Marketing` |
| Any fatal flaw at "kills the deal" severity (wrong ICP, no budget exists, blocker confirmed permanent, contact left the company, etc.) | `Disqualify` |

### Output

Emit two pieces of state for downstream steps:
- `mqlToSqlDisposition` — one of `Convert MQL→SQL` / `Hold in Marketing` / `Disqualify`.
- `mqlToSqlRationale` — 1–2 sentences naming the specific gates that passed/failed with one-line evidence each (e.g., "Authority Fail: contact is a researcher in discovery, not a buyer. Timeline Fail: 18+ month build/pivot horizon.").

### CRM write implications — recommendation only, never an automatic move

**The MQL→SQL disposition is advisory. It does NOT trigger any pipeline or stage write.** Existing deals are never moved automatically by this routine (see Step 6). Whatever the disposition, the deal's `pipeline` and `dealstage` are left exactly as they are in the CRM.

The disposition and rationale are still computed and surfaced everywhere a human will read them — the CRM note, the Notion `MQL → SQL` column, and the Slack pill — framed as a recommendation for a human to action:
- `Convert MQL→SQL` → recommend a human promote the deal to the qualification pipeline (suggested target: `pipeline` `<PROSPECT_PIPELINE_ID>`, `dealstage` `<PROSPECT_QUALIFICATION_STAGE_ID>`). Do not write it.
- `Hold in Marketing` → no recommended move.
- `Disqualify` → recommend a human disqualify (suggested target: `dealstage` `<MARKETING_DISQUALIFIED_STAGE_ID>`). Do not write it.

## Step 6 — Write to the CRM (auto-push, pre-authorized)

Pre-flight:
- Tool: `mcp__<CRM_MCP_ID>__manage_crm_objects` — must be AVAILABLE
- Account: <CRM_OWNER_NAME> (`<CRM_OWNER_EMAIL>`), ownerId `<CRM_OWNER_ID>`, accountId `<CRM_ACCOUNT_ID>`
- Confirm write permissions for: NOTE, DEAL, CONTACT, COMPANY, CALL, MEETING_EVENT

Always call `get_user_details(include=["TOOL_INFORMATION"])` once at start to re-confirm permissions haven't changed. If `manage_crm_objects` is no longer AVAILABLE, abort writes and fall back to producing a strict JSON payload + deal URL in the Slack summary.

**Critical: this routine runs unattended.** Every `manage_crm_objects` call must include `confirmationStatus: "CONFIRMATION_WAIVED_FOR_SESSION"` since there is no human to approve the inline confirmation prompt. Auto-push was pre-authorized during routine setup. **If you have NOT explicitly opted into unattended CRM writes, run in `--dry-run` mode instead (see Manual invocation) — it stops before any write.**

### Owner lookup
Resolve the host's CRM ownerId via `search_owners(searchQuery="<HOST_1>")` or `search_owners(searchQuery="<HOST_2>")`. Cache the two owner IDs across runs. Stamp the note's `hubspot_owner_id` with that owner — not with the connector's own account (`<CRM_OWNER_ID>`).

### Note creation
Use `manage_crm_objects` with `createRequest`. Note body shape:
- `<h3>{Company} — Post-Call Audit ({Meeting Date})</h3>`
- **If** the deal is in the Marketing pipeline (`<MARKETING_PIPELINE_ID>`): an `<h4>MQL → SQL Disposition</h4>` block immediately under the H3, containing the disposition value + the rationale produced in Step 5b. Omit this block entirely for non-Marketing pipelines.
- `<pre>{recap text}</pre>`
- `<p><a href='{recordingUrl}'>Call recording</a></p>`

```
{
  "confirmationStatus": "CONFIRMATION_WAIVED_FOR_SESSION",
  "createRequest": {
    "objects": [{
      "objectType": "notes",
      "properties": {
        "hs_note_body": "<h3>{Company} — Post-Call Audit ({Meeting Date})</h3>[<h4>MQL → SQL Disposition</h4><p><b>{disposition}</b> — {rationale}</p> if Marketing pipeline]<pre>{recap text}</pre><p><a href='{recordingUrl}'>Call recording</a></p>",
        "hs_timestamp": "{meeting end time as epoch ms}",
        "hubspot_owner_id": "{host owner ID}"
      },
      "associations": [
        {"targetObjectId": {dealId}, "targetObjectType": "DEAL"},
        {"targetObjectId": {contactId}, "targetObjectType": "CONTACT"}
      ]
    }]
  }
}
```

### Deal field updates
Use `manage_crm_objects` with `updateRequest`. Only include properties that actually change.

**Never write `pipeline` or `dealstage` on an existing deal.** This routine does not move deals automatically — pipeline and stage are owned by a human. The Stage Action and MQL→SQL dispositions from Step 5/5b are surfaced as *recommendations only* (in the note, Notion, and Slack); they are never written to the CRM. This applies regardless of disposition value or which pipeline the deal is in.

(The single exception is a brand-new deal created in Step 4 because none existed — that sets `pipeline`/`dealstage` at *creation* time, which is not a move.)

Deal fields this routine DOES write (these are not "moves"):
- `amount` — only if changed.
- `closedate` — only if the call surfaced an explicit timeline change (ISO date string).
- `description` — append the dated audit summary to existing description; do not overwrite. Read current value first via `get_crm_objects`, then concatenate `\n\n--- Auto-audit {YYYY-MM-DD} ---\n{summary}`. When an MQL→SQL disposition was produced (any value), prefix the appended summary with `[MQL→SQL recommendation: {disposition}]` so the description history carries the trail.

Always include `confirmationStatus: "CONFIRMATION_WAIVED_FOR_SESSION"`.

> **CRM validation gotcha:** some pipelines/stages require custom fields on a
> `description` (or other) write and will reject it, even though a NOTE write to
> the same deal succeeds. If a deal-field write returns a validation error,
> capture it, keep the successful note, mark the row "Manual Review", and do NOT
> retry the failing write.

If any write returns a permission error: capture the failure, fall back to producing a strict JSON payload and the deal URL in the Slack summary for manual entry. Do not retry destructive writes.

## Step 7 — Append a row to the Notion database

Read `summary-target.txt` in this skill folder for the Notion data source URL.

Use `mcp__<DATABASE_MCP_ID>__notion-create-pages` to create one page in the **Morning Call Audits** data source (`<NOTION_DATA_SOURCE_URL>`). Populate every property the audit produced:

- `Audit` (title): `{Company} — Post-Call Audit ({Meeting Date})`
- `Meeting Date` (date)
- `Company`, `Prospect`, `Host Attendee`, `Tag`, `Stage Action`
- `MQL → SQL` (select: `Convert MQL→SQL` / `Hold in Marketing` / `Disqualify`) — **only set this property when the deal is in the Marketing pipeline** (`<MARKETING_PIPELINE_ID>`). For deals in any other pipeline, omit the property entirely from the create-pages payload (do not send a null or empty value — leave the field unset).
- `Close Probability` (number, 0–100)
- `Current Amount`, `Recommended Amount` (dollar)
- `Primary Use Case`, `Required Products` (multi-select — only emit values that appear in the schema)
- `Budget`, `Authority`, `Need`, `Timeline` (each: Present / Partial / Missing)
- `Latent Pain`, `Active Pain`, `Quantified Impact`, `Solution Dev`, `Evaluation`, `Decision`, `Objection Handling` (0–10 numbers)
- `Fatal Flaws`, `Illusion Signals`, `Top 3 Next Questions`, `Top 3 Executive Moves` (rich text, newline-separated bullets)
- `Executive Recap` (rich text, ≤150 words)
- `CRM Deal` (URL), `Call Recording` (URL)
- `Writes Status` (multi-select, mark which CRM writes succeeded; include `Manual Review` if any failed)

Inside the new page body, paste the full fenced text recap inside a code block so the page itself is the human-readable artifact (matches what was pushed to the CRM).

If a `Required Products` value the audit recommends doesn't already exist in the schema (e.g., a new product), drop it from the multi-select and note it in `Executive Recap` instead — do not attempt to mutate the schema at runtime.

## Step 8 — Send the Slack summaries

### Channel preflight (hard requirement)

The destination channel ID is fixed: **`<SLACK_CHANNEL_ID>`** (`<SLACK_CHANNEL_NAME>`), declared in `summary-target.txt` as `slack_channel_id`.

1. Read `slack_channel_id` from `summary-target.txt` at the start of Step 8 and bind it to a local variable. Do not skip this read even if you "remember" the ID from earlier in the run.
2. Assert `slack_channel_id == "<SLACK_CHANNEL_ID>"`. If it does not match, **abort the post** for that call — do not fall back to a guessed channel — and surface a manual-review entry.
3. The `channel` field on every Slack API call below MUST be the string literal `"<SLACK_CHANNEL_ID>"`. Do **not** derive it from: the current conversation's thread context, the user's recent Slack activity, a channel mentioned earlier in the run, the prospect's category, or any other inferred signal. Pass this exact ID every time, on every call.
4. Posts must use the Morning Call Scoring bot (`<SLACK_BOT_APP_ID>`) via `curl` + `$<SLACK_BOT_TOKEN_ENV>`. Do **not** use a Slack MCP send-message tool — it triggers Slack's consecutive-author grouping that this routine is explicitly designed to avoid.

### Why the channel is hard-pinned

Inferring the Slack `channel` from run context (e.g. "the deal is in the
Marketing pipeline, so post to the marketing channel") will drift posts to the
wrong channel. Always read the ID from `summary-target.txt` and pass that exact
literal — never any other value. **Keep a dated log of any drift incident
here.**

### Configuration source

Post via the **Morning Call Scoring** Slack bot (app id `<SLACK_BOT_APP_ID>`) — NOT via a Slack MCP. Posting as the bot gives every audit its own avatar + name in the channel, which prevents Slack from grouping consecutive audits into a single visual stack.

The bot's OAuth token lives in `.claude/settings.local.json` env as `<SLACK_BOT_TOKEN_ENV>` (gitignored — never commit it). Post via `curl` against the Slack Web API:

```bash
curl -sS -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $<SLACK_BOT_TOKEN_ENV>" \
  -H "Content-type: application/json; charset=utf-8" \
  -d '{"channel":"<SLACK_CHANNEL_ID>","text":"<message body>"}'
```

The response includes a `ts` field — capture it from the top-level post and pass it as `thread_ts` in the follow-up reply:

```bash
curl -sS -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $<SLACK_BOT_TOKEN_ENV>" \
  -H "Content-type: application/json; charset=utf-8" \
  -d '{"channel":"<SLACK_CHANNEL_ID>","thread_ts":"<top-level-ts>","text":"<reply body>"}'
```

Send **two posts per scored call** — a **top-level message** containing only the top blockquote, and a **thread reply** containing the recap + metrics + BANT + links. Process calls in start-time order. Two `curl` calls per audit (top-level first, then reply).

When building the JSON payload in shell, use a heredoc + `jq -Rs` (or python) to safely escape the message body — never inline a multi-line string with quotes/emoji directly into `-d`, that breaks for any non-trivial body (and emoji can trigger shell "character not in range" errors that half-post). Recommended pattern:

```bash
BODY=$(cat <<'EOF'
> 🦊 🦊 🦊
> ...full message body here, raw...
EOF
)
PAYLOAD=$(jq -n --arg ch "<SLACK_CHANNEL_ID>" --arg t "$BODY" '{channel:$ch, text:$t}')
curl -sS -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $<SLACK_BOT_TOKEN_ENV>" \
  -H "Content-type: application/json; charset=utf-8" \
  --data "$PAYLOAD"
```

The split:

**Top-level message** = section 0 only (the top blockquote) plus the trailing spacer.

**Thread reply** = sections 1–4 (executive recap, key metrics, BANT, links) plus the optional `⚠️` line if CRM writes were skipped. No trailing spacer needed in the reply.

Each call's content includes ONLY these sections, in this order:

0. **Top blockquote (emoji + header + participants):** wrap the first three pieces of content in a Slack blockquote — every line of the section begins with `> `. The blockquote contains:
   - **Emoji divider** — 3 copies of the same emoji, drawn at random from a standard emoji category. Pick a different emoji per call so adjacent messages don't collide.
   - One blank quoted line (`>` alone).
   - **Header line** — `*Morning audit — {Company} ({Meeting Date})*` with the deal URL.
   - One blank quoted line.
   - `*Participants*` followed by one bulleted line per attendee (name + role/org). List every attendee from the meeting payload — host-side AND prospect attendees, including no-shows if the recorder flagged them.

   Example top:
   ```
   > 🦊 🦊 🦊
   >
   > *Morning audit — Acme (2026-05-04)* · <link|CRM deal>
   >
   > *Participants*
   > • <HOST_1> (<YOUR_COMPANY> — AE, host)
   > • Jane Buyer (Acme — VP Risk)
   ```

   End the blockquote with a non-quoted blank line, then continue with the rest of the message in regular formatting.
1. **Executive recap (2 sentences):** a tight 2-sentence distillation of the longer Executive Recap. Operator voice, no marketing tone.
2. **Key metrics — emoji pills before each metric (red 🔴 / yellow 🟡 / green 🟢):**
   - **MQL → SQL** — only include this line when the deal is in the Marketing pipeline (`<MARKETING_PIPELINE_ID>`). Omit the line entirely for any other pipeline. Pill: 🟢 Convert MQL→SQL, 🟡 Hold in Marketing, 🔴 Disqualify. Follow the line with one short sub-line of rationale: gates passed (e.g., "1/4 SQL gates · Auth Fail, Timeline Fail, Pain Fail, Blocker Fail").
   - Current amount — pill is 🟡 (neutral baseline).
   - Recommended amount — 🟢 if increased, 🔴 if decreased, 🟡 if maintained.
   - Close probability — 🔴 if <35%, 🟡 if 35–65%, 🟢 if >65%.
   - Stage action — 🟢 Forward, 🟡 Stay, 🟡 Backward, 🔴 Disqualify.
3. **BANT — emoji pill before each line:**
   - 🟢 Present, 🟡 Partial, 🔴 Missing — followed by one-line evidence per the recap.
4. **Links:** CRM deal · Call recording · Notion audit row.
5. **Trailing spacer:** end every message body with three non-breaking-space spacer lines so the bubble has visible breathing room before the next message in the channel. Concretely: append `\n\n \n \n ` to the very end of the message body. Plain `\n\n\n` does NOT work — Slack collapses consecutive blank lines unless each line carries a non-whitespace character (nbsp counts).

If CRM writes were skipped or any operational error occurred for a given call, append one trailing line inside that call's block prefixed with `⚠️` linking to the Notion row for manual review (place it BEFORE the trailing spacer). Do not expand into a section.

Do NOT include: writes-performed checklist, top 3 executive moves, top 3 questions, fatal flaws, illusion signals — those live in the Notion row only.

### Posting order

Each call gets its own top-level Slack message — do not concatenate. Post in start-time order (earliest call first). Zero qualifying calls → post a single line "No qualifying calls for {date}" and exit.

## Step 9 — Idempotency

Run idempotency **per call**, before that call's writes. Check both:
1. The deal's existing notes for one whose body starts with `{Company} — Post-Call Audit ({Meeting Date})`.
2. The Notion `Morning Call Audits` data source via `notion-search` filtered by `data_source_url=<NOTION_DATA_SOURCE_URL>` and the audit title.

**Run BOTH checks before ANY write.** The CRM note search can lag behind a
concurrent run, so relying on it alone has produced duplicate audits — the
Notion-row check is the backstop. If either exists for a given call, skip writes for that call and surface "Already scored — see existing audit" in that call's Slack block (still include the block so the morning recap is complete). Do not skip the entire run because one call was already scored.

## Failure handling

- Network / MCP tool failure: retry once after 30s. If still failing, log to summary and exit.
- Ambiguous deal match: skip the write, surface for manual review in summary.
- Transcript not yet ready in the recorder (sometimes lags): retry once after 5 min. If still missing, log "Transcript not ready — will retry on next run" and exit.
- Never bypass CRM validation errors with force-style flags.

## Manual invocation

When invoked manually (`/score-morning-call` or via direct skill call), accept optional args:
- `--date YYYY-MM-DD` to override the prior-business-day default
- `--meeting-id <recorder-id>` to score a specific meeting
- `--dry-run` to produce the recap + proposed updates and Slack summary, but skip CRM writes

In manual mode with `--dry-run`, also render the visualizer widget per the rubric's output format and stop before writes for human approval. **If you have not opted into unattended auto-push, always use `--dry-run`.**
