---
sidebar_position: 91
title: "Kanban Dispatcher Liveness"
description: "Ready-age watchdog, dispatcher heartbeat, and poison-task quarantine so a stalled dispatcher can never silently idle ready work"
---

# Kanban Dispatcher Liveness

A task once sat in `ready` for about 30 minutes with no dispatcher process running, and
nothing paged the board. Every stuck signal that exists today is either computed on pull or
lives inside the dispatcher process itself. The result was a silent stall: ready work idle
for the length of the incident. This page specifies the three mechanisms that make that
exact stall impossible to miss, each with its verification command and threshold
justification.

## What exists today

- **Stats age line.** `hermes kanban stats` (`hermes_cli/kanban.py` `_cmd_stats` L1143)
  prints `Oldest ready task age: {age}s` (L1155–1157) from `board_stats`
  (`hermes_cli/kanban_db.py` L4197–4222). Computed on pull, not alerting, and it measures
  `created_at`, not ready-transition age.
- **Diagnostics.** `hermes_cli/kanban_diagnostics.py` `_rule_stranded_in_ready` (L682–736)
  already flags assigned, unclaimed ready tasks older than `stranded_threshold_seconds`
  (default 30 min, `DEFAULT_CONFIG` L754–765) with an escalation ladder (L712–718). But it
  is pull-only: `_cmd_diagnostics` (L635–710) prints and returns 0 unconditionally. Nothing
  pushes a page.
- **In-process stuck warnings.** The deprecated daemon's `HEALTH_WINDOW = 6`
  (`hermes_cli/kanban_ops.py` L204) and the gateway watcher's mirrored `_HEALTH_WINDOW = 6`
  (`gateway/kanban_watchers.py` L37, warn at L309–315) warn when ready work goes unspawned
  for 6 consecutive ticks. Both are emitted by the dispatcher, the one process whose absence
  is the incident.
- **Tick telemetry.** Per-board isolation in `gateway/kanban_watchers_dispatcher.py`
  `tick_once_for_board` (L174–207) and spawn summaries in `_log_spawn_results` (L319–334).
  The dispatch-tick hook `on_kanban_dispatch_tick` (`kanban_db.py` L239, fired at
  `kanban_db_dispatch.py` L1963/1974) is in-process best-effort with no persisted state.
  There is no periodic liveness record readable from `list`/`stats`.
- **Poison-task breaker.** `DEFAULT_FAILURE_LIMIT = 2` (`kanban_db_dispatch.py` L34–36).
  `_record_task_failure` (L1336–1456) flips the task to `blocked` and appends `gave_up`
  (L1420–1455). `recompute_ready` (`kanban_db.py` L2128) skips blocked-at-limit rows, so the
  park is durable and only `unblock_task` exits it.

## Spec 1: ready-age watchdog

Rule: page from outside the dispatcher process whenever an assigned, unclaimed, spawnable
ready task's ready-transition age exceeds N = 900 s, escalating the page with age and with
heartbeat staleness.

Where it lives: a new `hermes kanban watchdog` subcommand (`_cmd_watchdog` in
`hermes_cli/kanban.py` beside `_cmd_stats`). It runs board-side with no dispatcher alive,
which is the exact failure mode it must catch. Per-task readiness reuses
`_rule_stranded_in_ready`'s basis (`kanban_diagnostics.py` L682–707). The spawnability
filter reuses `has_spawnable_ready` (`kanban_db_dispatch.py` L1745) plus
`check_respawn_guard` (L1491): guard-held work (rate-limit cooldown 300 s, PR-window hold)
reports as `held`, never as the page driver. The threshold home is
`stranded_threshold_seconds` in `kanban_diagnostics.DEFAULT_CONFIG`, retuned 1800 to 900 so
one knob serves both the pull rule and the paging watchdog. Page shape: stdout
`PAGE ready_stall <task_id> age=<s>s severity=<warning|error|critical>` with exit 2
(0 = healthy), run by a launchd/systemd timer. Severity ladder: warning at N, error at 2N or
when the heartbeat (spec 2) is simultaneously STALE, critical at 6N.

Verification command (acceptance bar): `hermes kanban watchdog --ready-age-threshold 60`
against a scratch board holding one assigned, unclaimed ready task older than 60 s and no
dispatcher running must return exit 2 with the `PAGE ready_stall` line naming that task.
With the task claimed, or a respawn-guard event newer than the age window, the same command
returns 0 and lists the task under `held`. The override flag is test-only. Production reads
`stranded_threshold_seconds`.

Why N = 900 s rather than the incident's 30 minutes:

1. 30 min is the incident length, not a bound derived from the system. A page at 30 min
   arrives only once the damage equals the observed stall. N = 900 pages at the incident's
   midpoint, and the error step (2N = 1800 s) lands where the old silent threshold sat.
2. N is 15 default dispatch intervals (60 s). The in-repo rationale for 1800 s budgeted 30
   ticks of claim latency against a 1-tick event. 15 ticks is already generous.
3. N exceeds the in-process stuck window (6 ticks, about 6 min at the default interval), so
   an alive-but-stuck dispatcher warns first and the external page is the backstop.
4. N bounds every legitimate transient below it: gateway restart boot delay, the 300 s
   rate-limit cooldown, and retry spacing via `check_respawn_guard`.

## Spec 2: dispatcher liveness heartbeat

Rule: record one heartbeat row per board on every completed locked dispatch tick, and
surface its age in `hermes kanban stats`, marked STALE past a freshness bound of F = 3 times
`kanban.dispatch_interval_seconds` (180 s at the default 60 s).

Where it lives: writer in `kanban_db_dispatch.py` `dispatch_once` (L1919–1975), a one-row
UPSERT into a new `dispatcher_heartbeats(board TEXT PRIMARY KEY, tick_at INTEGER NOT NULL,
pid INTEGER, ticks INTEGER)` table, appended on locked-tick completion beside the WAL
checkpoint and deliberately not on the `skipped_locked` branch, preserving the invariant
that a losing dispatcher writes nothing. Both drivers pass through `dispatch_once` every
tick (gateway `tick_once_for_board`, standalone `run_daemon`), so one write site covers the
production loop and the deprecated daemon alike. Reader: `_cmd_stats` gains one line under
the ready-age line, `Dispatcher heartbeat: <age>s ago (board <slug>, pid <p>)[, STALE]`,
and the dashboard reads the same row. A heartbeat event kind in `task_events` was rejected:
the table is task-scoped (`task_id TEXT NOT NULL`), and a task-less tick event would cost
one insert per tick per board (about 1,440 rows/day at 60 s) against one UPSERT row.

Verification command (acceptance bar): `hermes kanban stats | grep 'Dispatcher heartbeat'`
while a dispatcher ticks must show a fresh line naming the board and pid. Negative proof:
stop the dispatcher in a scratch environment, re-run after F has passed, and the line must
read STALE.

Why F = 3 times the interval (floor 120 s): one loop pass is the interval plus per-tick
work, boards tick serially, and the pass includes zombie reap and auto-decompose. Two
missed ticks plus slack absorb one slow multi-board pass and one gateway restart. Past
3 times the interval no dispatcher is ticking the board. F scales with the configured
interval. A stricter bound false-alarms on one slow pass. A looser one re-creates the silent
window.

## Spec 3: poison-task quarantine

Rule: park any task in `blocked` with `gave_up` after `failure_limit` = 2 consecutive
non-success spawn attempts, never auto-resume it without an operator, and never let it stop
the tick loop.

The mechanism exists and this spec locks it: the breaker chain above, durable through
`recompute_ready`, with tick continuity from per-board isolation and the watchers'
one-tick-fails-never-stops-the-next error handling (`gateway/kanban_watchers.py` L256–257,
L321–324). No product-code change is required. The build lane lands the probe experiment as
the gate test:

1. On a scratch board, create a probe task with `skills=["nonexistent-skill-sentinel"]`
   (exactly one skill name, so the zero-resolve raise is deterministic) and a control task
   with no skill requirement.
2. Let the dispatcher tick. Every probe spawn exits non-success with
   `Error: Unknown skill(s): nonexistent-skill-sentinel`; after the second attempt the
   breaker trips.
3. Read-back: `hermes kanban events <probe-id>` must show exactly `spawned`, `spawned`,
   `gave_up` with payload `{failures: 2, effective_limit: 2, limit_source: "dispatcher",
   trigger_outcome: "crashed"}`, and `hermes kanban list --status blocked` must list the
   probe. A manual dispatch pass must not promote the probe.
4. Tick-continuity proof: on the tick after the trip, the gateway log shows a spawn summary
   line for the control task, and across all probe ticks there is no
   `kanban dispatcher: unexpected watcher error` and no
   `kanban dispatcher: tick failed on board <board>`.

Why the threshold is 2: one non-success is routine flakiness. Two consecutive non-successes
with no success in between (the counter resets only on success, `_clear_failure_counter`
L1476–1479) is the smallest budget that separates poison from flakiness and hard-bounds a
poison task's worker burn to 2 spawns. Escape hatches stay: per-task `max_retries` override
wins, infrastructure refusals never count, and systemic/terminal-provider failures trip
faster.

## Adoption order

1. Spec 2 (heartbeat) first: the smallest additive change (one table, one UPSERT, one stats
   line), and the other surfaces read it.
2. Spec 1 (watchdog) second: the subcommand, the 1800 to 900 threshold retune, and the
   timer unit.
3. Spec 3 (quarantine proof) last: as the build's gate test against the built dispatcher on
   a scratch board.
