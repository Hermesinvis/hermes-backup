Server: Linux, UTC timezone (Iran = UTC+3:30). Port 22 blocked — use HTTPS for git. Hermes binary: /opt/venv/bin/hermes. Telegram chat_id: 8352373787.
§
Amir's GitHub backup repo: Hermesinvis/hermes-backup. Backup script lives at ~/.hermes/scripts/hermes-backup.sh. Cron job ID for backup: 72ebedcdac70 (daily at 07:00 UTC = 10:30 IRDT). Cron job ID for fun facts: bbd02be1515d (every 2 minutes).
§
Hermes cron delivery works via Telegram adapter — messages delivered to chat_id 8352373787. When user is actively chatting, cron messages may appear delayed or buried in the DM flow. Ticker heartbeat can go stale after gateway restarts; restart gateway to fix.
§
Backup script at /data/.hermes/scripts/hermes-backup.sh backs up memories, skills, config, SOUL.md to GitHub repo (Hermesinvis/hermes-backup) via HTTPS. Git global user: Hermes Backup / hermes-backup@bot. Port 22 blocked on this host — use HTTPS only.