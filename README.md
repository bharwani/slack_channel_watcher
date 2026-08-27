# slack-sev-monitor

Repo copy of a Claude Code skill that monitors the Fastly `#ic` Slack
channel for new SEV1/SEV2 incident messages and DMs a summary when one
is found.

## Layout

```
slack-sev-monitor/
├── SKILL.md      # skill definition — the logic Claude follows on each run
├── README.md     # skill-specific docs (files, scheduling, changing channel)
└── state.json    # runtime state snapshot (last-checked ts, notified ids, cached Slack user id)
com.fastly.slack-sev-monitor.plist   # launchd job that fires the skill every 15 minutes
```

This is a copy for reference/version control. The live, running copies
that Claude Code and launchd actually use are
`~/.claude/skills/slack-sev-monitor/` and
`~/Library/LaunchAgents/com.fastly.slack-sev-monitor.plist`.

## How it works

On each run: reads new `#ic` messages since the last check, filters for
`SEV1`/`SEV2` (case-insensitive), pulls thread context for matches,
redacts secrets found in message text, and DMs one summary per new
incident to the user. No matches means no DM. See
`slack-sev-monitor/SKILL.md` for the full step-by-step logic and
`slack-sev-monitor/README.md` for scheduling details (it runs every 15
minutes via a macOS launchd LaunchAgent, independent of any open Claude
Code session).

## Keeping this copy in sync

`state.json` here is a point-in-time snapshot, not live — the real
state lives at `~/.claude/skills/slack-sev-monitor/state.json` and
updates on every scheduled run. If you edit `SKILL.md` here, copy it
back to `~/.claude/skills/slack-sev-monitor/SKILL.md` for the change to
take effect.

Likewise, editing `com.fastly.slack-sev-monitor.plist` here has no
effect on the schedule until you copy it back to
`~/Library/LaunchAgents/com.fastly.slack-sev-monitor.plist` and reload
it:

```sh
cp com.fastly.slack-sev-monitor.plist ~/Library/LaunchAgents/
launchctl unload ~/Library/LaunchAgents/com.fastly.slack-sev-monitor.plist
launchctl load ~/Library/LaunchAgents/com.fastly.slack-sev-monitor.plist
```
