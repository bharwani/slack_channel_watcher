# slack-sev-monitor

Claude Code skill that sweeps the Fastly `#ic` Slack channel for new
SEV1/SEV2 incident messages and DMs a summary to the user when found.

## How it works

- Invoke on demand with `/slack-sev-monitor`, or let the scheduled job
  (below) run it automatically.
- Optionally pass a number of days to review, overriding the normal
  incremental window: `/slack-sev-monitor 3` (or headlessly `claude -p
  "/slack-sev-monitor 3"`) reviews the last 3 days regardless of the
  last checkpoint. Dedup via `state.json` still prevents re-sending DMs
  for incidents already flagged. With no argument, behavior is
  unchanged (incremental since last check, or last 24h on first run).
- State is tracked in `state.json` in this directory: last-checked
  timestamp, already-notified message timestamps (dedupe), and the
  user's cached Slack ID.
- Each run: reads new `#ic` messages since the last check, filters for
  `SEV1`/`SEV2` (case-insensitive, `sev-1`/`sev 2` etc. all match),
  pulls thread context for matches, redacts any secrets found in the
  message text, and DMs one summary per new incident. No matches means
  no DM.

See `SKILL.md` for the full step-by-step logic.

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | Skill definition and instructions Claude follows on each run |
| `state.json` | Runtime state — last checked timestamp, notified message IDs, cached Slack user ID |
| `README.md` | This file |

## Scheduling

Runs automatically every 15 minutes via a macOS launchd LaunchAgent —
not the in-session `CronCreate` tool, so it keeps running independently
of any open Claude Code session and survives reboots.

- Plist: `~/Library/LaunchAgents/com.fastly.slack-sev-monitor.plist`
- Invokes `claude -p /slack-sev-monitor` headlessly with a narrow
  `--allowedTools` scope so it never blocks on an interactive permission
  prompt. The exact granted set (must match what `SKILL.md` actually
  calls):
  `Read Write Edit mcp__fastly__current_time mcp__claude_ai_Slack__slack_read_channel mcp__claude_ai_Slack__slack_read_thread mcp__claude_ai_Slack__slack_search_users mcp__claude_ai_Slack__slack_send_message`
- Logs: `~/Library/Logs/slack-sev-monitor.log` and `.err.log`.

### Manual/headless testing

Any manual invocation of `claude -p "/slack-sev-monitor"` — including
from another directory, like the repo copy in `~/Downloads/ty` — needs
the same `--allowedTools` grant, or it'll block waiting for interactive
approval that a non-interactive session can't give:

```sh
claude -p "/slack-sev-monitor" --allowedTools "Read Write Edit mcp__fastly__current_time mcp__claude_ai_Slack__slack_read_channel mcp__claude_ai_Slack__slack_read_thread mcp__claude_ai_Slack__slack_search_users mcp__claude_ai_Slack__slack_send_message"
```

`SKILL.md` deliberately hardcodes the `#ic` channel ID and uses
`mcp__fastly__current_time` for "now" instead of `slack_search_channels`
or `Bash`/`date` — keep it that way, since adding either back would
require widening this allowlist again.

Manage the schedule:

```sh
# stop
launchctl unload ~/Library/LaunchAgents/com.fastly.slack-sev-monitor.plist

# restart
launchctl load ~/Library/LaunchAgents/com.fastly.slack-sev-monitor.plist

# check it's loaded
launchctl list | grep slack-sev-monitor
```

## Changing the monitored channel

Edit the channel reference in `SKILL.md` (currently `#ic`,
`C036MF29X`). Also clear `state.json` back to defaults if switching
channels, so the first run re-bounds its lookback window instead of
using a stale timestamp from the old channel.
