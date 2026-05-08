---
name: watchers
description: Poll RSS, JSON APIs, and GitHub with watermark dedup.
version: 1.0.0
author: hermes
license: MIT
metadata:
  hermes:
    tags: [cron, polling, rss, github, http, automation, monitoring]
    related_skills: [github-hermes-agent-dev]
---

# Watchers — interval-polling with dedup

Cron already runs things on a schedule. This skill is the pattern for turning "poll an external source every N minutes and tell me when something new shows up" into a cron job, using three ready-made scripts that handle the tedious part: tracking what you've already seen so you don't re-deliver the same items every tick.

**Mental model.** Watchers are just cron jobs in `no_agent=True` mode with a polling script that:

1. Fetches data from the external source
2. Compares against a watermark file of previously-seen IDs
3. Writes the new watermark back
4. Prints new items to stdout (or nothing — empty stdout = silent, existing cron behavior)

That's the whole thing. The scripts below do all three; you just wire them into cron.

## Ready-made scripts

Three scripts, all in `scripts/`. Each reads `WATCHER_STATE_DIR` env var (defaults to `$HERMES_HOME/watcher-state/`) to find its watermark file, keyed by the `--name` argument.

| Script | What it watches | Dedup key |
|---|---|---|
| `scripts/watch_rss.py` | RSS 2.0 or Atom feed URL | `<guid>` / `<id>` |
| `scripts/watch_http_json.py` | Any JSON endpoint returning a list of objects | Configurable id field |
| `scripts/watch_github.py` | GitHub issues / pulls / releases / commits for a repo | `id` / `sha` |

All three:
- First run records a baseline — never replays existing feed
- Watermark is a bounded ID set (max 500) to cap memory
- Output format: `## <title>\n<url>\n\n<optional body>` per item, suitable for direct delivery
- Empty stdout on no-new — cron will not deliver (silent by design)
- Non-zero exit on fetch errors — cron records the error

## Setup — wire a watcher into cron

Pick the script, pick a name (used for the watermark filename), pick an interval + delivery target.

**Example 1: RSS feed → deliver to Telegram every 15 minutes**

```bash
hermes cron create my-rss \
  --schedule "*/15 * * * *" \
  --no-agent \
  --script "$HERMES_HOME/skills/devops/watchers/scripts/watch_rss.py" \
  --script-args "--name hn --url https://news.ycombinator.com/rss --max 5" \
  --deliver telegram
```

**Example 2: Watch GitHub issues on a repo, deliver to the origin chat**

```bash
hermes cron create hermes-issues \
  --schedule "*/5 * * * *" \
  --no-agent \
  --script "$HERMES_HOME/skills/devops/watchers/scripts/watch_github.py" \
  --script-args "--name hermes-issues --repo NousResearch/hermes-agent --scope issues" \
  --deliver origin
```

Set `GITHUB_TOKEN` in `~/.hermes/.env` to avoid the 60 req/hr anonymous rate limit.

**Example 3: Poll an arbitrary JSON API**

```bash
hermes cron create api-events \
  --schedule "*/1 * * * *" \
  --no-agent \
  --script "$HERMES_HOME/skills/devops/watchers/scripts/watch_http_json.py" \
  --script-args "--name api --url https://api.example.com/events --id-field event_id --items-path data.events" \
  --deliver origin
```

## Delivering verbatim vs handing to the agent

- `--no-agent` (all examples above): script's stdout is delivered verbatim as the message. Zero LLM cost. Good for digests and notifications where you just want the raw list.
- **Omit `--no-agent`**: the script's stdout becomes the agent's prompt context via `context_from`. Use this when you want the agent to reason over the new items — summarize, prioritize, decide whether to wake you up, etc. Costs tokens per tick; only use when you want the reasoning.

## State files

Every watcher writes `$HERMES_HOME/watcher-state/<name>.json` — one file per named watcher. Inspect with:

```bash
cat $HERMES_HOME/watcher-state/my-rss.json
```

To force a replay (treat the next run as a first poll), delete the state file:

```bash
rm $HERMES_HOME/watcher-state/my-rss.json
```

## Writing your own

All three shipped scripts use the same template: load watermark → fetch → diff → save watermark → emit new items. `scripts/_watermark.py` is the shared helper; import it from a custom script to get atomic writes + bounded ID set + first-run baseline handling for free. See any of the three reference scripts for how little boilerplate it takes.

## Pitfalls

- **Every script MUST print nothing on empty delta.** cron treats empty stdout as "don't deliver"; if your script prints a "no new items" header you'll spam the channel every tick. The shipped scripts handle this; check yours does too.
- **The first run is always silent.** This is deliberate — recording a baseline instead of replaying the entire feed. If you need an initial digest, add a `--prime-with-latest N` flag in your own script, or just check the state file after the first run and manually emit once.
- **Watermark files grow boundlessly if you don't cap.** `_watermark.py`'s helper caps at 500 IDs. Raise the cap if your feed has high churn; lower it if you're on a tight filesystem.
- **Don't put the watermark in a directory the agent's sandbox can't write to.** `$HERMES_HOME/watcher-state/` is always writable; custom paths may not be from inside Docker/Modal.
