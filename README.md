# cache-warm

A Claude Code slash command that keeps a running session's prompt cache
warm during a break (lunch, a meeting) instead of letting it expire and
paying a full rebuild cost on your next message.

Claude Code's prompt cache TTL resets on every cache **hit** — a genuine
API request that reuses the cached prefix. There's no other way to keep a
cache warm; a local timestamp file touches nothing server-side and does
nothing. This command uses `ScheduleWakeup` to fire that request on a
timer, from inside the session that's already running, without you typing
anything.

## Install

Copy the command into your Claude Code commands directory:

```bash
cp commands/cache-warm.md ~/.claude/commands/cache-warm.md
```

## Usage

- `/cache-warm` — start warming (default: 50-minute ping interval, 3h
  auto-stop window)
- `/cache-warm 2h` — start warming with a custom total window
- `/cache-warm off` — stop immediately

## Notes

- Each ping is a real, billed API request (mostly cheap cache-read
  pricing) that appends a small turn to the session transcript.
- Only works while the Claude Code process stays running and the machine
  stays awake.
- Only one `ScheduleWakeup` slot exists per session — starting
  `/cache-warm` while an unrelated `/loop` is active will replace it.
- Only useful on API-key / token-metered billing. On Pro/Max subscription
  plans your quota is gated by request count, not token cost — every ping
  eats your 5-hour quota for no benefit. Don't use this on a subscription
  plan.
- This relies on a documented but not contractually guaranteed property
  of Anthropic's prompt cache (a cache read resets the TTL). It could
  change without notice.

## Related work

[yujiachen-y/claude-code-cache-keepalive](https://github.com/yujiachen-y/claude-code-cache-keepalive)
solves the same problem and is worth a look — same underlying trick
(a cache read resets the TTL), different shape:

- **Mechanism**: it hooks `Stop` and sleeps inside the hook process
  (`sleep interval && emit {"decision":"block",...}`), which re-injects a
  synthetic user turn. This command instead uses `ScheduleWakeup`, which
  hands control back to the harness between pings rather than blocking a
  hook process — nothing needs to sleep, and it doesn't need you to hit
  `Esc` to get your prompt back while it's running.
- **Distribution**: it's a full plugin + marketplace
  (`/plugin marketplace add` then `/plugin install`), with a config UI,
  env var overrides, and a log file. This is one markdown file copied into
  `~/.claude/commands/` — no plugin system involved.
- **Target TTL**: it's tuned for the 5-minute default TTL (240s interval,
  capped at ~28 min of coverage per idle gap, resetting on your next real
  message). This is tuned for the 1-hour extended-cache TTL, meant for one
  long, deliberate break (default 3h window) rather than covering gaps
  between messages.
- **Transcript**: it sends a real (if bland) user-style message each ping
  and Claude replies to it, visible in the transcript. This command's
  pings carry no user-facing text — just a `ScheduleWakeup` call, so
  nothing shows up as a fake exchange.
