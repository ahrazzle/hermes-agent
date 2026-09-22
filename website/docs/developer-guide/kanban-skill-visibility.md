---
sidebar_position: 90
title: "Kanban Task Skill Visibility"
description: "Validate kanban task skill requirements against the assignee profile at create time, with the spawn-time unknown-skill raise kept as the backstop"
---

# Kanban Task Skill Visibility

A kanban task that names skills in its `skills` field only has those skills checked for
syntax at create time. Whether they actually resolve under the **assignee's** profile is
discovered at dispatch. When nothing resolves, the worker raises before spawn and the
dispatcher counts it as a crash. A task that could never run is therefore retried to the
failure limit and auto-blocked, instead of being refused the moment it was created with an
error naming the missing skill and the assignee.

This page documents the mechanism and the recommended rule: validate requested skills
against the assignee profile at create time, and keep the spawn-time raise unchanged as the
backstop.

## Incident

A task was created with `skills=["protean-stoa-exchange"]` for an assignee whose profile had
an empty `skills/` directory and no `skills.external_dirs` entry covering the skill's home.
The task crashed twice in a row with worker output
`Error: Unknown skill(s): protean-stoa-exchange`, after which the dispatcher gave up
(`gave_up`, `failures=2`, `effective_limit=2`, `limit_source=dispatcher`) and the task sat
blocked until a manual unblock after the profile was repaired. Nothing in that chain was a
resolver bug: the condition was knowable at create time and no check looked for it.

## Mechanism (create → dispatch → resolve → raise)

1. **Create.** `hermes_cli/kanban_db.py` `_normalize_task_skills` (L1209–1246), called from
   `create_task` (L1315): strips, dedupes, refuses commas (L1224–1228) and toolset names
   (L1229–1245). There is no existence check against any skill root and no assignee-profile
   resolution: any non-toolset string is stored on the task row.
2. **Dispatch.** `hermes_cli/kanban_db_dispatch.py` `_default_spawn` (L2761) binds the worker
   to the assignee profile (`normalize_profile_name` / `resolve_profile_env`, L2775/L2781);
   `_worker_argv` (L2673) emits one `--skills X` pair per stored name verbatim (L2686–2690).
3. **Resolve.** The worker boots with `HERMES_HOME` set to the assignee profile home
   (`hermes_cli/main.py` L616–629) and resolves the flags in `hermes_cli/oneshot.py`
   `_build_preloaded_skills_prompt` (def L124, called L568) via
   `agent/skill_commands.py` `build_preloaded_skills_prompt` (L610). Skill roots come from
   `agent/skill_utils.py` `get_all_skills_dirs` (L398): the profile `skills/` dir,
   `skills.create_dir`, and `get_external_skills_dirs` (L337–363: validated
   `skills.external_dirs`, `~`/`${VAR}`-expanded, existing directories only).
4. **Raise.** `hermes_cli/oneshot.py` L133–142: the `ValueError(f"Unknown skill(s):
   {missing_display}")` fires only when `loaded_skills` is empty, i.e. **zero** requested
   skills resolved. Partial misses warn and continue
   (`logging.warning("Unknown skill(s) requested, skipping: %s. Continuing with: %s. ...")`),
   matching the CLI chat partial-success contract (`cli.py` L1013–1020).
5. **Outcome.** The `ValueError` surfaces as worker output `Error: Unknown skill(s): ...`
   and the dispatcher counts it as a crash toward the run limit.

## Options considered

| # | Candidate rule | Failure mode | Cost |
|---|---|---|---|
| A | Create-time validation: reject create when a requested skill does not resolve under the assignee profile's skill roots. | Post-create profile drift can still fail at spawn (caught by the unchanged backstop). Over-rejects if the skill lands between create and spawn, but the error is immediate and re-create is cheap. | One resolver call at create plus one shared helper |
| B | Warn-and-continue at spawn: downgrade the zero-resolve `ValueError` to a warning. | A mandated skill silently absent produces non-compliant worker output with no spawn-time signal. Deletes the fail-loud invariant. | About one line |
| C | Dispatcher pre-spawn check plus quarantine: do not spawn, auto-block (`needs_input`) naming the missing skill instead of crash-counting. | Detection one stage later than A on the same resolver seam, plus new quarantine state and unblock semantics to maintain. | Medium: dispatcher hook plus block path |

## Recommended rule

At task creation, reject any requested skill that does not resolve under the assignee
profile's skill roots (its `skills/` directory, `skills.create_dir`, and
`skills.external_dirs`), failing fast with the missing names and the assignee. Keep the
spawn-time `ValueError` (raise only when `loaded_skills` is empty) unchanged as the backstop.

Why A beats the alternatives: in the incident the failing condition (the assignee's roots
not containing the skill) was knowable at create time. Option A turns two crash cycles plus
an auto-block plus a long silent stall into one actionable create-time error naming the
assignee, the skill, and the root to fix. Option B would have produced worker runs without
their mandated skill procedures, a silent quality and compliance loss worse than a stall.
Option C adds state machinery for a seam A already needs and only buys detection margin.

TOCTOU is inherent: resolution state is mutable (the assignee's profile changed mid-incident
in the field case), which is exactly why the spawn-time raise stays authoritative and A is an
early-fail gate, not a replacement.

## Adoption sketch

- Hook: `hermes_cli/kanban_db.py` `create_task` at the skills normalization call (L1315;
  `_normalize_task_skills` L1209–1246). After normalization, resolve each name under the
  assignee's profile home and raise a create error listing the missing names and the assignee
  (message shape mirrors `oneshot.py` L136).
- Resolver: reuse `agent/skill_commands.py` `build_preloaded_skills_prompt` (L610) resolution
  with roots from `agent/skill_utils.py` `get_all_skills_dirs` (L398) /
  `get_external_skills_dirs` (L337–363), pinned to the assignee home through the existing
  home-override seam (`hermes_constants.set_hermes_home_override`, as used by
  `build_auto_load_prompt`, `agent/skill_commands.py` L660–665). No second implementation of
  resolution.
- Unchanged: `kanban_db_dispatch.py` `_worker_argv` (L2673) / `_default_spawn` (L2761), the
  oneshot raise/warn contract (L124–143), crash counting, auto-block, and all gates.

## Operational check (available today, no code change)

Before leaving a task with an explicit skills list in `ready`, and before retrying a
skill-crashed task, confirm each requested skill resolves under the assignee profile:

```bash
HERMES_HOME="$HOME/.hermes/profiles/<assignee>" hermes skills list | grep -c <skill>
```

Each requested skill must appear at least once. A dispatch whose assignee resolves none of
the requested skills is invalid: fix the profile (`skills.external_dirs`) or the task's
skills field before dispatching, never retry past the raise.
