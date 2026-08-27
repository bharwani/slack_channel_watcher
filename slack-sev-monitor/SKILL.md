---
name: slack-sev-monitor
description: Checks the #ic Slack channel for new SEV1/SEV2 incident messages and DMs a summary when found. Use when asked to check for incidents, run a SEV sweep, or when invoked on a schedule to monitor #ic.
---

## Trigger and scope

Invoked on-demand (`/slack-sev-monitor`) or on a schedule (launchd, every
15 minutes) to sweep the `#ic` Slack channel for new SEV1/SEV2 incident
messages and DM the user a summary. Idempotent — safe to run repeatedly;
state on disk prevents duplicate notifications.

State file: `~/.claude/skills/slack-sev-monitor/state.json`

```json
{ "last_checked_ts": null, "notified_ts": [], "self_slack_id": null }
```

## Steps

1. **Load state.** Read `state.json`. If missing, create it with the
   defaults above.

2. **Resolve own Slack ID (once).** If `self_slack_id` is null, call
   `mcp__claude_ai_Slack__slack_search_users` for `abharwani@fastly.com`
   and cache the returned user ID into `self_slack_id`.

3. **Read new channel messages.** Call
   `mcp__claude_ai_Slack__slack_read_channel` for `#ic`.
   - If `last_checked_ts` is null (first run), bound the lookback to the
     last 24 hours instead of the full channel history.
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

7. **Update state.** Set `last_checked_ts` to the current time. Append
   each notified message's timestamp to `notified_ts`, keeping only the
   most recent 500 entries. Write `state.json` back out.

8. **No matches.** If no new SEV1/SEV2 messages are found, send no DM.
   Still update `last_checked_ts` in state.
