---
name: hermes-automation
description: "Cron jobs, backup scripts, and scheduled message delivery."
version: 1.1.0
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
- **Persian/Farsi text in prompts triggers injection filter.** The `cronjob_manage` tool blocks prompts containing U+200C (zero-width non-joiner), which is essential for correct Persian rendering. Always write cron prompts in English and instruct the agent to output in the user's language. The cron agent runs fresh each tick and follows the prompt — a language instruction in English works fine.
- **`continuity: true` causes exponential prompt bloat.** Each run nests the previous run's full output (including that run's nested output) inside the prompt. At frequent intervals (e.g., every 5m), context grows exponentially and buries the actual prompt instructions. Disable continuity for content-delivery jobs where deduplication is not needed: `cronjob_manage(action='update', job_id='...', continuity=False)`. Only enable continuity for jobs that genuinely need to see their own history (scouts, incremental digests).
- **Agent content jobs must explicitly require tool use.** Without an explicit instruction like `You MUST use web_search`, the cron agent falls back to its own memory and produces stale, repeated, or fabricated content. Always include a concrete tool-use directive in the prompt for any job that needs fresh/real data per run.