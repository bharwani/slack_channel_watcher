---
name: slack-sev-monitor
description: Checks the #ic Slack channel for new SEV1/SEV2 incident messages and DMs a summary when found. Use when asked to check for incidents, run a SEV sweep, or when invoked on a schedule to monitor #ic.
---

## Trigger and scope

Invoked on-demand (`/slack-sev-monitor`) or on a schedule (launchd, every
15 minutes) to sweep the `#ic` Slack channel for new SEV1/SEV2 incident
messages and DM the user a summary. Idempotent — safe to run repeatedly;
state on disk prevents duplicate notifications.

**Optional argument — lookback days.** Accepts a single optional integer
argument giving the number of days to review, e.g. `/slack-sev-monitor 3`
or headlessly `claude -p "/slack-sev-monitor 3"`. When given, it
overrides the normal incremental window (see step 3) and reviews the
last N days regardless of `last_checked_ts` — useful for an ad hoc
manual sweep. `notified_ts` dedup still applies, so re-running over an
already-covered range won't re-send DMs for incidents already flagged.
With no argument, behavior is unchanged (incremental since
`last_checked_ts`, or last 24h on first run).

State file: `~/.claude/skills/slack-sev-monitor/state.json`

```json
{ "last_checked_ts": null, "notified_ts": [], "self_slack_id": null }
```

Channel: `#ic`, ID `C036MF29X` (hardcoded — do not look it up via
`slack_search_channels`; that tool isn't in this skill's granted set and
will block headless/scheduled runs).

Current time: call `mcp__fastly__current_time` to get "now" wherever
these steps say so (do not shell out via `Bash`/`date`; that tool isn't
in this skill's granted set either).

All reads/writes of `state.json` must use the `Read`/`Write`/`Edit`
tools directly — never `Bash` (e.g. `cat`, `echo >`, a script). This
skill's granted tool set doesn't include `Bash`, and shelling out to
touch a path outside the invoking session's working directory triggers
an extra sensitive-path approval prompt that blocks headless/scheduled
runs.

## Steps

1. **Load state.** Read `state.json`. If missing, create it with the
   defaults above.

2. **Resolve own Slack ID (once).** If `self_slack_id` is null, call
   `mcp__claude_ai_Slack__slack_search_users` for `abharwani@fastly.com`
   and cache the returned user ID into `self_slack_id`.

3. **Read new channel messages.** Call
   `mcp__claude_ai_Slack__slack_read_channel` for channel ID `C036MF29X`.
   - If a lookback-days argument was given, bound the lookback to `now -
     N days` (via `mcp__fastly__current_time`), ignoring `last_checked_ts`.
   - Otherwise, if `last_checked_ts` is null (first run), bound the
     lookback to the last 24 hours instead of the full channel history.
   - Otherwise only consider messages newer than `last_checked_ts`.

4. **Filter for SEV1/SEV2.** Case-insensitive match against each
   message's text for `\bsev[-\s]?1\b` or `\bsev[-\s]?2\b`. Skip any
   message whose timestamp is already in `notified_ts`.

5. **Summarize each match.** If the message is a thread parent, fetch the
   thread with `mcp__claude_ai_Slack__slack_read_thread` for context.
   Build a short summary: severity (SEV1/SEV2), affected
   service/incident title, current status, key timeline points, and a
   link back to the message/thread. Redact any secrets, tokens, or
   credentials found in the message text with `[REDACTED]` before
   including it — never forward raw secrets into the DM.

6. **Notify.** Send the summary as a Slack DM to `self_slack_id` via
   `mcp__claude_ai_Slack__slack_send_message`. One DM per new
   incident found, not one giant digest.

7. **Update state.** Set `last_checked_ts` to the current time (via
   `mcp__fastly__current_time`),
   regardless of whether a lookback-days argument was used — this keeps
   subsequent incremental (no-argument) runs continuing forward from
   now, rather than replaying the wider window again. Append each
   notified message's timestamp to `notified_ts`, keeping only the most
   recent 500 entries. Write `state.json` back out using the `Write` or
   `Edit` tool (not `Bash`).

8. **No matches.** If no new SEV1/SEV2 messages are found, send no DM.
   Still update `last_checked_ts` in state.
