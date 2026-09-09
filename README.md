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
