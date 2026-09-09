---
name: cache-warm
description: "Keep this session's prompt cache warm during a break by pinging it on a timer — real API requests that hit the cache and reset its TTL, not a local heartbeat. Usage: /cache-warm [duration] to start (default 3h window, 50m interval), /cache-warm off to stop."
license: MIT
---

# /cache-warm — keep the prompt cache warm during a break

The user is stepping away (lunch, a meeting) and wants this session's prompt
cache to survive the gap, so their next real message doesn't pay a full
cache-rebuild cost. Claude Code's prompt cache TTL resets on every cache
**hit** — a genuine API request that reuses the cached prefix. There is no
other way to keep a cache warm; a local timestamp file (what some third-party
tools do) touches nothing server-side and does not work. This command uses
`ScheduleWakeup` to make this exact session fire that request on a timer,
without the user typing anything, for as long as the process stays alive and
the machine stays awake.

## Parsing `$ARGUMENTS`

- `off` or `stop`: call `ScheduleWakeup` with `stop: true`. Tell the user
  cache-warming is stopped. Do nothing else.
- A duration like `2h`, `90m`: use it as the total warming window.
- No arguments: default to a **3h** window.

## Starting the loop

1. Compute `end_at` = now + the window, as a Unix timestamp.
2. Interval: default **50 minutes** (3000s). Claude Code's prompt cache TTL
   is a fixed 1h on this account (seen live in the statusline payload's
   `prompt_cache.ttl`), so 50m gives a flat 10-minute safety margin without
   pinging any more often than necessary.
3. Tell the user, briefly: cache-warming is on, pinging every `<interval>`,
   auto-stopping around `<end_at, local time>` unless they run
   `/cache-warm off` first. Mention that if an unrelated `/loop` is already
   running in this session, starting this will replace it (only one
   `ScheduleWakeup` slot exists per session).
4. Call `ScheduleWakeup`:
   - `delaySeconds`: the interval in seconds
   - `noop`: false
   - `reason`: `"cache-warm: keeping session prefix hot during a break"`
   - `prompt`: the following text verbatim, with `<END_AT>` and
     `<INTERVAL_SECONDS>` filled in. This string is what you will see,
     unadorned, when woken — it must be fully self-contained since a long
     gap may have compacted away these instructions from context:

     ```
     <<cache-warm-ping>> end_at=<END_AT> interval=<INTERVAL_SECONDS>

     This is an automated cache-warm ping (see /cache-warm in
     ~/.claude/commands/cache-warm.md). Do not message the user. Do not call
     any tool other than ScheduleWakeup.

     If now >= end_at: call ScheduleWakeup with stop: true. Say nothing —
     the warming window ended on schedule, this is not worth interrupting
     the user for.

     Otherwise: call ScheduleWakeup again with delaySeconds=interval,
     the exact same prompt text (unchanged end_at/interval), noop: true,
     reason "cache-warm: periodic ping".
     ```

## On wake (a prompt starting with `<<cache-warm-ping>>`)

Follow the embedded instructions in that prompt exactly — they are the
source of truth (this section may not be in context anymore by then). The
API request that delivers this wake-up is itself the cache-warming action;
there is nothing else to do beyond deciding whether to continue or stop.

## Notes

- Each ping is a genuine, billed API request (mostly cheap cache-read
  pricing) that appends a small turn to the session transcript — not free,
  but tiny relative to a full cache rebuild.
- Only works while this process keeps running and the machine stays awake.
  If the machine sleeps past the TTL, the cache will already have expired by
  the next ping — it just resumes normal warming from there, no error.
