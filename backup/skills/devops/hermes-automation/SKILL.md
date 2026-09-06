---
name: hermes-automation
description: "Cron jobs, backup scripts, and scheduled message delivery."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [cron, automation, backup, scheduling, telegram]
    category: devops
    related_skills: [hermes-agent, github]
---

# Hermes Automation Workflows

Procedures for cron jobs, backup scripts, scheduled delivery, and GitHub integration. Covers the Hermes cron scheduler, script-only jobs, agent-based jobs, and platform delivery.

## Cron Jobs — Setup Procedure

1. **Decide job type:** script-only (`no_agent: true`, runs a shell/Python script, delivers stdout) vs agent-based (LLM generates a response from a prompt, can use tools like `web_search`).
2. **Create the job:**
   ```python
   tool_call(name='cronjob_manage', arguments={
       'action': 'create',
       'schedule': 'every 2m',  # or cron expr '0 7 * * *'
       'name': 'Human-friendly name',
       'deliver': 'origin',  # sends to the chat where the job was created
       'prompt': '...',  # REQUIRED for agent jobs
       'script': 'path/to/script.sh',  # for script-only or pre-run data
       'no_agent': True,  # only for script-only jobs
   })
   ```
3. **Verify delivery:** After creation, run `cronjob_manage(action='list')` and confirm `deliver` target resolves. Then fire a manual test: `cronjob_manage(action='run', job_id='...')`.

### Scheduling Formats
- Interval: `"every 2m"`, `"every 1h"`, `"every 30m"`
- Cron expression: `"0 7 * * *"` (daily at 07:00 UTC)
- Natural: `"every monday 9am"`, `"weekdays at 9am"`

### Pitfalls

- **Attached skills block the job if env vars are missing.** The cron preflight validates all attached skills before running. If a skill needs `$SOME_API_KEY` and it's not set, the job shows `blocked_config` and never fires. Remove the skill from the job: `cronjob_manage(action='update', job_id='...', skills=[])`.
- **Delivery conflicts with active session.** When the user is actively chatting, cron delivery may get queued or delayed. The gateway logs will show `Normal final-send NOT suppressed despite active stream consumer`. This is a known Hermes behavior — the cron output is generated but delivery is deferred until the session settles.
- **Gateway restart kills the cron ticker.** After `hermes gateway restart`, verify the ticker is alive: `cat ~/.hermes/cron/ticker_heartbeat` and compare timestamp to `date +%s`. If stale by >5 minutes, the ticker isn't running.
- **Timezone mismatch.** The Hermes cron system runs in UTC. For Iran time (UTC+3:30), 10:30 IRDT = 07:00 UTC. Use cron expressions in UTC, not local time.
- **Git operations need HTTPS when port 22 is blocked.** Embed the token in the clone URL: `https://TOKEN@github.com/user/repo.git`. Set `git config --global user.name/email` before first commit.
- **`no_agent` jobs require `script`.** Without a script, the job has nothing to run and silently does nothing. Agent jobs require `prompt`.

## Backup Script Pattern

For backing up Hermes data to GitHub:

```bash
#!/bin/bash
set -e
REPO_URL="https://TOKEN@github.com/user/repo.git"
BACKUP_DIR="/tmp/hermes-backup-repo"
HERMES_HOME="${HERMES_HOME:-$HOME/.hermes}"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S %Z')

# Clone or pull
cd "$BACKUP_DIR" 2>/dev/null && git pull --rebase origin main || \
  git clone "$REPO_URL" "$BACKUP_DIR" && cd "$BACKUP_DIR"

# Copy critical data
mkdir -p backup/memories backup/skills
cp -r "$HERMES_HOME/memories/"* backup/memories/ 2>/dev/null || true
cp "$HERMES_HOME/config.yaml" backup/ 2>/dev/null || true
cp "$HERMES_HOME/SOUL.md" backup/ 2>/dev/null || true
find "$HERMES_HOME/skills" -name "SKILL.md" -exec bash -c '
    dest="backup/skills/${0#$HERMES_HOME/skills/}"
    mkdir -p "$(dirname "$dest")"
    cp "$0" "$dest"' {} \;

# Commit if changed
git add -A
git diff --cached --quiet || (git commit -m "Auto-backup: $TIMESTAMP" && git push)
```

Place scripts in `~/.hermes/scripts/` and reference them in cron jobs by relative path.

## Agent-to-Telegram Direct Delivery (Fallback)

When `deliver: origin` fails to reach Telegram (e.g., session conflict), bypass the Hermes delivery layer by sending directly via Telegram Bot API:

```python
import requests, json, os
BOT_TOKEN = open(f'{os.path.expanduser("~")}/.hermes/.env').read()\
    .split('TELEGRAM_BOT_TOKEN=')[1].split('\n')[0].strip()
CHAT_ID = '8352373787'  # from ~/.hermes/channel_directory.json
requests.post(f'https://api.telegram.org/bot{BOT_TOKEN}/sendMessage',
    json={'chat_id': CHAT_ID, 'text': message, 'parse_mode': 'Markdown'})
```

Use this only as a fallback when Hermes delivery consistently fails.