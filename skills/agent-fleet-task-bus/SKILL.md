---
name: agent-fleet-task-bus
description: Operate a multi-container Hermes agent fleet sharing a filesystem task bus — health checks, the dispatcher, model pinning, cron delivery, and the failure modes that look like success. Use for agent control rooms, agent fleets, task queues for agents, or "my agents aren't doing work".
license: MIT
---

# Agent fleet + task bus operations

Running N agent containers that claim work from a shared filesystem queue. The hard
part is not starting them — it's knowing whether they are actually doing work.

## When to use

- "Are the agents actually doing anything?" / tasks piling up unclaimed
- A fleet container answers `No LLM provider configured`, 401, or 429
- Provisioning a new domain agent, or a watchdog that logs errors every tick
- One provider key across the whole fleet hitting a quota cap

## The four layers

A healthy container is not a working fleet. Check in order, because each layer can
pass while the next is broken:

1. **Containers up.** `docker ps --format '{{.Names}}' | grep -c '^agent-'` — non-zero
   and stable against your health script's own count. Read the count from `docker ps`,
   never hardcode it in a check.
2. **Health timer green.** The periodic health script logs `all N healthy`. If it logs
   a different N than step 1, the script is stale.
3. **Dispatcher active.** `systemctl list-timers <dispatch>.timer`. The bus consumer
   must be running, not merely installed.
4. **Inbox draining.** `bus status` — inbox should trend to zero.

If 1–3 pass and 4 shows a growing inbox, **the bus is write-only**: nothing claims
work. That is the most important failure to distinguish, because everything looks
green from the outside.

## The dispatcher

A timer-driven script that, per queue with inbox files: claims one task (flock),
runs the agent non-interactively with a timeout, then moves the agent's output
document to outbox and archives the task. Sequential by design — N containers on
0.5 CPU each will not parallelize usefully.

Prompt invariants to bake into the dispatcher, because the agent will otherwise
interpret the task file literally:

- **Never send email, post, or submit to any portal.** The agent drafts; a human sends.
- **Always write the deliverable to the outbox path**, regardless of the path the
  task file names. See the safe-root pitfall below.
- `status: blocked` with a specific question **counts as success**. Otherwise an agent
  that correctly cannot proceed retries forever or fabricates an answer.

## Pitfalls

- **`HERMES_WRITE_SAFE_ROOT` silently swallows task-file writes.** Tasks say "draft
  into `bus/tasks/<queue>/working/pending/`", but the bus mount sits outside the safe
  root, so `write_file` is denied — and the run still reports success with an empty
  outbox. The dispatcher prompt must explicitly override the output path.
- **The outbox directory must exist and be owned by the container's uid.** The agent
  runs unprivileged (commonly uid 10000); a root-owned directory fails at the atomic
  temp-file rename with `Permission denied`, after the agent has done all the work.
- **All containers sharing one provider key means one quota cap stalls the entire
  fleet simultaneously.** Free tiers do not sustain a fleet: a free-model daily cap or
  a 20-requests/day tier dies in minutes. Use a flat-rate provider for fleet work.
- **A model swap is three writes, not one:** the provider pin, the provider config
  block, and the API key in the container environment. Missing the env var and
  missing the config block produce the *same* error (`No LLM provider configured`), so
  check both before debugging further. Make the swap script idempotent.
- **`rc=0` on a rate-limit failure.** The non-interactive agent exits 0 even when the
  provider 429s. **Trust the presence of the output file, not the exit code.** This is
  the single most misleading signal in the whole system.
- **Some containers predate the fleet layout.** A deploy script that assumes one
  compose path will skip them; patch those by hand rather than "fixing" the script
  mid-run.
- **A fresh container needs auth *and* config.** Seeding only `config.yaml` leaves it
  at "No LLM provider configured" because credentials live in a separate `auth.json`.
  Copy both from a known-good sibling, plus `SOUL.md` and `skills/`, then
  `chown -R <uid>:<uid>`.
- **Read a container's model pin before diagnosing it.** One grep over every container
  and `sort | uniq -c` on the provider column is faster than N individual smoke tests,
  and a whole fleet on one provider is the expected state, not a bug.
- **Sibling containers drift.** After any bulk config pass, smoke several, not one.
  `admin` working says nothing about `db` — separate files, and loops skip some.
- **Do not diagnose a stalled bus from `journalctl` alone.** `systemctl is-active`
  reports `activating` while a task is mid-run, which reads as hung but is normal.
  Check the per-task run log: growing means working, zero-byte means it just started.

## Watchdog bug pattern

```sh
# WRONG: yields "0\n0" when grep matches nothing (grep prints 0 AND exits 1, then echo 0 appends)
X=$(cmd | grep -c pat || echo 0)

# RIGHT
X=$(cmd | grep -c pat || true)
```

Symptom: `[: 0\n0: integer expression expected` in the journal every tick, with the
mitigation branch never running. A watchdog that errors every tick is worse than no
watchdog — it trains you to ignore the log.

## Cron delivery needs a resolved target, not just a token

`deliver: telegram` alone fails with "no delivery target resolved" even when the bot
token is set. Two prerequisites:

1. The allowed-users list must be **non-empty** — an empty value boots fine and
   resolves no DM.
2. The channel directory must be populated, which happens only once a DM actually
   arrives.

Verify with a direct send to an explicit `platform:<chat_id>` target, then edit the
cron job to point at that same target, then force one run.

## Host layout that works

| What | Path |
|---|---|
| Bus | `/srv/agent-bus` (mounted `/opt/bus`) |
| Per-agent compose | `/srv/fleet/<name>.yml` |
| Legacy compose | `/srv/<name>/docker-compose.yml` |
| Container data | `/srv/<name>/data` → `/opt/data` |
| Dispatcher | `/usr/local/bin/<dispatch>`, log `/var/log/<dispatch>.log` |
| Per-task run log | `/var/lib/<dispatch>/run.<task>.log` |

When installing a helper script over SSH: write it locally, `bash -n` / `py_compile`
it, then stream it with `cat file | ssh host 'cat > /usr/local/bin/x && chmod +x x'`.
**Windows CRLF survives that pipe and breaks the shebang on the host** — strip it
first. Verify with `bash -n` on the remote side, not the local one.
