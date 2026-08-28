# Guided Onboarding — the welcome flow

The first-run chain: **cinematic → solo welcome chat with Hermes → the user's
first task, built live in its own session.** No wizard window, no sign-in card,
no survey for its own sake. The user walks out having *done something*, with a
build in their sessions list and a guide they can come back to.

The guided chat is not an anonymous session. It belongs to a persistent
**`hermes-setup` profile**, as an ordinary visible chat titled
`Welcome to Hermes`. It arranges the app, decides the first task with the user,
and then hands it off — it does not build the task itself. It stays alive as
training wheels: it hears how the handoff went, and its check-ins ride the
build's own progress.

**`setup` is an internal name only.** It is the profile key, the atoms, and the
hidden `[setup]` note marker. The user never reads it: the voice introduces
itself as *Hermes* and nothing else, and the script forbids "Setup", "the setup
assistant", "the onboarding guide", and the whole not-really-the-agent framing.
There is one Hermes; the welcome chat is where they meet it.

**There is no bot surface anywhere in this flow.** The guide is a profile with
a normal chat, the first build is a normal session on the user's `default`
profile, and assembly keeps the Bots pane out of the tree entirely. Nothing
here mints a bot, stamps `ui_meta['hermes-bots']`, or creates a hidden
canonical chat.

## Where things live

| Piece | Path |
| --- | --- |
| The model's runbook (the whole script) | `../../store/onboarding-script.ts` |
| Profile, persona, fast lane, seed messages | `setup-profile.ts` |
| Greeting bank, solo mode, layout assembly | `assembly.ts` |
| Which step renders what (`::onboarding`) | `directive.tsx` |
| The cards themselves | `cards/{setup,build,first-screen}.tsx` |
| Check-in pacing, off real work done | `first-build.ts` |
| Kickoff + handoff + check-in delivery | `../../app/contrib/onboarding-handoff.ts` |
| The parting signpost (accent tour step) | `signpost.ts` |
| The cloud-or-local offer (both pills) | `../../store/suggestion-providers/hermes-account.ts` |
| The local-model download (a stand-in) | `../../store/suggestion-providers/local-model.ts` |

## Script (the model's runbook)

Lives in `store/onboarding-script.ts::buildChatOnboardingPrompt` — keep this
document in sync with it.

**The card beats carry no tool calls, and memory is written once.** This is
RULE 4, and it is load-bearing rather than tidy. The runbook used to say
"persist what matters THE MOMENT it lands… never end a turn still owing one",
which on a fast model turns a returning tool result into a cue to speak and
save again: a live run answered the name "brooke" with six API calls, six
identical memory writes and thirty-six seconds of "Brooke it is." before
stalling short of the colour card. The `::onboarding` directives already
persist name, colour, connectors and layout, so the beats need no tool at all,
and the one memory write moves to the handoff turn where every fact is known.
Do not reintroduce a per-beat save.

0. **The banked greeting** — the opening line is pre-written: seeded as a
   real assistant row at `session.create` (so the chat rehydrates with it) AND
   typed in client-side like a streamed turn, so the first paint is instant and
   alive. The runbook tells the model it already spoke; its first real turn
   answers the user's name. The guided turns ride the fast lane (see
   `FAST_LANE` in setup-profile.ts) pinned on the `hermes-setup` profile — the
   user's real default model is untouched, and nothing they run Hermes as
   elsewhere reaches in here. The composer hides its model pill on this thread
   for the same reason it hides the git strip: it names a lane we chose.
1. **Name** — "What should I call you?" — saved via the invisible
   `::onboarding{step="name" value="…"}` data directive.
2. **Theme** — `::onboarding{step="look"}` (accent pick, live retint).
3. **Connectors** — `::onboarding{step="connectors"}` (tools they use; stored,
   not wired).
4. **Layout** — `::onboarding{step="layout"}` (the app assembles around the
   chat).
5. **The look around** — `::ask{question="Want a look around first?"
   options="Just the basics|Show me around|I'll figure it out"}`. The first two
   both run the built-in `tour` tool — `action="targets"` first to see what is
   actually on screen, then one `action="start"` built only from what it
   reported — and differ in depth: "Just the basics" is three steps on the
   essentials (conversations, composer, new chat), "Show me around" is 4–6.
   "I'll figure it out" is one line and on. Whichever they pick, the turn
   closes by saying the tour is always on offer.

    Basics used to be three lines of prose and no overlay. Describing a button
    is worse than pointing at it: the user still has to go find the thing, and
    the words cost the same turn the highlight would have.

    What the tour can point at is `data-tour` handles on the chat surfaces
    (`app/chat/tour-marker.ts`) — sidebar, new chat, composer, send, model pill,
    transcript, terminal, settings, profile rail. They go to the PRIMARY chat
    only: session tiles mount the same tree, and a handle on all of them makes
    the collector discard the target rather than pick one.

    Lightest first, deliberately: the three-line version should be the one
    their eye lands on, so the full tour reads as a step up rather than the
    default. Nobody wants to open a new app into a click-through.

    This beat sits AFTER the layout pick on purpose. Before it the window is
    the conversation and nothing else, so a tour would highlight a chat pane
    and stop.
6. **The fork** — one sentence, then `::ask{question="Know what you'd like it
   to make?" options="…" input="true"}` — clickable pills, typed answers
   welcome. The options come from `forkOptions()`, because the machine decides
   the shape (see **The machine fork**).

### Branch A — they have something in mind (general or specific)

7a. If general or not-sure: ask **what they're working on right now** — saved
    with `::onboarding{step="working" value="…"}`, which writes the same
    `context` answer the rest of the flow reads. Then surface **generated
    options** as tappable chips — `::onboarding{step="first" options="…|…|…"}`
    — 2–4 options spanning simple (a reminder) to complex (a dashboard), built
    from that answer plus their tools. If specific: skip both and go straight
    to the handoff.

    **Every branch is NO-AUTH, not local-only.** The first build must need
    zero external accounts — but the browser, web search, scripts, and
    computer use are all fair game, and the more VISIBLE the better. Never
    Gmail/Slack/Google sign-in: connectors get wired later, on the user's
    request. If their idea needs one, shape the task around its no-auth core
    and name the connection as a later step.
8a. **The handoff.** The chip tap (or their message) decides the task. Hermes
    replies with ONE short framing sentence and emits
    `::onboarding{step="handoff" task="…" brief="…"}` — it does NOT start the
    work. The renderer opens a session on the user's `default` profile, seeds
    it with the work-side runbook, moves the user into it, and submits the
    brief as the user's first visible turn. The build starts from that turn.
9a. **Permissions note** (build session, one short sentence as work begins):
    "I'll ask for permissions as we go — say no to anything."
10a. **Progress artifact** — `::onboarding{step="progress"}` renders a
    live-updating card in the build session's transcript while it runs.

### Branch B — not sure

Same question as 7a, asked the way a stuck user can answer it: "what do you
wish you spent less time doing on the computer?" One follow-up to get concrete,
then the options card.

### Branch C — set up this machine

The machine itself is the job. Hermes asks ONE question (what they mainly want
the machine for) and hands off with `plan="machine-setup"`. It does not plan
the setup or list what it would install — the agent that takes the job audits
the box first.

### After the handoff — the welcome chat stays alive

- The renderer whispers a hidden `[setup] handoff complete` note into the
  welcome chat, which says one short line and stops.
- **The parting signpost** (`signpost.ts`). The handoff is the only moment in
  the run where the ground moves under the user — they were talking to Hermes
  on its own profile and they land mid-build in a session of their own — so the
  profile rail lights up once to say the chat they just had is still there. One
  accent-lit tour step, not a tour: the whole appeal of this flow is that it
  happens in conversation, and a click-through at the last beat would spend
  that. Skipped for the user who answered "I'll figure it out", read straight
  off the guide transcript before the switch replaces it.
- **The check-ins ride the build, not a clock** (`first-build.ts`). The guide used
  to schedule itself a daily cron here, which on a first run means a check-in
  that arrives tomorrow about a task that finished in four minutes. Instead the
  renderer counts the build's tool calls and, at two points, whispers a
  checkpoint note into that same session: the agent says in one line where the
  work stands and ends with an `::ask` for what the user wants next.
  - Only ever at a turn boundary (`message.complete`), because a note injected
    mid-loop is a synthetic user message inside an assistant turn.
  - Never stacked under a turn that already asked something.
- **Where should this run** rides the same counter: after ten real tool calls —
  past the first check-in, so they have seen the thing work and answered a
  "what next?" of their own — two pills appear in the composer's suggestion strip
  (`store/suggestion-providers/hermes-account.ts`), and only when the install is
  not already signed in. Sign in, or put a model on this machine. It used to be
  one pill on the reasoning that declining needs no click — true while "local"
  meant doing nothing, and false now that Hermes can offer to fetch the model.
  Both are real actions with a real cost, so both get a pill; ignoring the pair
  still leaves the user mid-task on whatever they already use.

  The download reports `progress` on the suggestion contract, so the pill fills
  itself rather than raising a bar that would have to explain which pill it
  belongs to. Sign-in reports nothing and keeps the spinner — an OAuth
  round-trip finishes when the user finishes. **The download is a stand-in**
  (`local-model.ts`): it reports time, not bytes. The offer, the fill, the
  cancel path and the failure path are all real, so landing the actual fetch is
  replacing one function body.

  Placeholder placement; the real anchor for the cloud-vs-local moment is still
  being found.
- If opening the build session fails, the whisper says so instead and Hermes
  builds the task in the welcome chat, so the flow never dead-ends.

## What the user walks away with

- A configured app (theme, layout, connectors noted) — the old wizard's job,
  done conversationally.
- A first task *started or built* — the competence moment.
- A guide they can always come back to, ignore, or retire. Training wheels.
- A mental model of how Hermes works: you describe a job, an agent takes it,
  it asks, it builds, it shows progress, it checks in.

## What deliberately does NOT happen

- **No login wall.** Inference is already configured (or the classic runtime
  check catches the first send). The chain never stops to authenticate.
- **No bot mode.** No minted task profile, no roster, no Bots tab. Assembly
  dismisses every pane the picked layout didn't declare, which is what keeps
  the left zone a plain sessions list instead of a two-tab strip.
- **No survey fatigue.** Every question either configures the app or feeds the
  first build. Nothing is collected "for later."
- **No nagging.** Two check-ins over a whole build, one account pill, and
  "stop checking in" stops it.

## Flow graph

```
                ┌─────────────┐
                │  cinematic  │  (intro reveal — welcome splash)
                └──────┬──────┘
                       ▼
                ┌──────────────────┐
                │  solo chat       │  small window, no sidebar/statusbar —
                │  = the welcome   │  a visible session titled
                │  1. name         │  "Welcome to Hermes", hermes-setup profile
                └──────┬───────────┘
                       ▼
              (2. theme → 3. connectors → 4. layout — the app assembles
               at the layout pick → 5. the look around)
                       ▼
                ┌─────────────┐
                │ 6. the fork │  "Know what you'd like it to make?"
                └──┬───┬───┬──┘
                   │   │   │
        ┌──────────┘   │   └──────────┐
        ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────────┐
  │ specific │  │ general  │  │  not sure    │
  │ in mind  │  │  idea    │  │  (probe)     │
  └────┬─────┘  └────┬─────┘  └──────┬───────┘
       │             ▼               │
       │      ┌───────────────┐      │
       │      │ 7a. generated │◀─────┘
       │      │ options card  │  ::onboarding{step="first" options="…"}
       │      └──────┬────────┘
       │             │ tap = the task is decided
       ▼             ▼
       ┌──────────────────────────┐
       │ 8a. HANDOFF              │  ::onboarding{step="handoff" task brief}
       │ session on `default`     │  renderer: seeded session.create →
       │ + the brief as turn one  │  switch → visible brief
       └───────┬─────────────┬────┘
               ▼             ▼
   ┌───────────────────┐   ┌────────────────────────┐
   │ BUILD session     │   │ SETUP's chat (alive)   │
   │ 9a. permissions   │   │ hidden [setup] note →  │
   │ 10a. progress     │   │ one goodbye-for-now    │
   │ the build runs    │   │ line, then quiet       │
   │ ↑ check-ins ride  │   └────────────────────────┘
   │   its tool calls  │
   └───────────────────┘
```

## The cards (transcript directives)

| step | attrs | renders | tap does |
|------|-------|---------|----------|
| `name` | `value="…"` | nothing | saves the name into the wizard answers |
| `look` | — | accent swatches | retints live, hidden `[setup]` report |
| `connectors` | — | connector chips | stored, hidden `[setup]` report |
| `layout` | — | layout previews | assembles the app live, hidden `[setup]` report |
| `working` | `value="…"` | nothing | saves what they're working on (same `context` field the options card is built from) |
| `first` | `options="A\|B\|C"` | generated chips | **visible user turn** — decides the task |
| `handoff` | `task="…" brief="…" plan="machine-setup"?` | a one-line status (opening → landed) | nothing to tap: the card raises the handoff beacon on mount |
| `progress` | `title="…"` | live build card (build session) | read-only; updates as the work streams |

Plus the general-purpose `::ask{question="…" options="A|B|C" input="true"}`
(ask-directive.tsx) — clickable pills for any fork; the pick is the user's
next visible turn.

Hidden `[setup]` reports are remembered (retry.ts): if a machine turn dies
before delivering anything, the report replays once, quietly — no red HTTP
row mid-setup. The skip affordance (skip.tsx) stays available throughout.

Reasoning text renders in **scratchpad** mode (`markdown-text.tsx`): no
artifact cards and no transcript directives. A model that thinks out loud about
the accent card used to render a live accent picker inside its own thinking
block.

The kickoff ADOPTS before it creates — an exact-title `session.list` lookup for
`Welcome to Hermes` on the setup profile resumes an existing chat (relaunch mid-onboarding,
dev re-kick) instead of forking a second one whose title stamp would silently
lose to the UNIQUE index and fall to the auto-titler.

The first-run frame: floating panes (user/plugin panels) stay hidden while solo
AND while the active session is any onboarding thread; the composer's git strip
is dropped on those threads too. Message-level chrome is suppressed by design
as well: the assistant action bar (branch-in-new-chat / copy / read-aloud /
regenerate, plus the reaction slot) and the turn-duration stamp are hidden on
onboarding threads via the `[data-thread-type='onboarding']` transcript hook in
styles.css — regenerate would re-roll a turn whose `::onboarding{…}` card the
step machine already consumed, and branching forks the user out of the guided
thread. Known caveat: `$chatOnboardingThreadIds` only holds session ids seen
this app-run, so a relaunch mid-onboarding rehydrates the welcome chat without the
marker and the chrome returns; extending the id set from the kickoff's adopt
path is deliberate follow-up work.

## The machine fork

Before the runbook is composed, the flow asks the host what it is
(`loadMachineProfile`, one IPC): platform, release, arch, whether there's an
NVIDIA GPU, the hardware's own model string, and how many days ago the OS
created this account.

Two things follow from it. **What the machine-setup option is called** — "Help
me set up this Mac", "…this PC", "…this Spark". And **whether it leads**: a
machine younger than three weeks, or a Spark at any age, gets that one option
plus "Something else", which opens the other four as a second ask. Four
alternatives beside the obvious answer is a menu; the obvious answer plus a way
out is an offer. Anything unknown counts as not-new — the option is always in
the list, it just doesn't lead without a reason.

Neither case is reachable from the machine you develop on, so the dev runner
can answer the probe for you: `npm run dev:fresh -- --new` overlays your own
host with an age of zero (a brand-new version of this Mac, this PC), and
`-- --spark` answers as an RTX Spark unboxed today. They overlay rather than
replace so `--new` stays your platform — the point of it is rehearsing "Help me
set up this Mac", which a wholesale fake payload can't give you. Your unmodified
host is the lived-in case.

The two Sparks are recognised differently because they are different computers.
An **RTX Spark** is a Windows-on-Arm PC (the N1X superchip, in this fall's
ASUS / Dell / HP / Lenovo / Surface / MSI laptops and mini desktops); the OEM
badge on the case isn't a name we can enumerate, so it's identified by shape —
Windows + Arm + NVIDIA silicon, which nothing else currently ships. The GPU
vendor comes from Chromium's own GPU enumeration (`app.getGPUInfo('basic')`, PCI
vendor `0x10DE`), so it's a lookup rather than a probe: no subprocess and no
vendor tooling a just-unboxed machine may not have yet. A **DGX Spark** is the
older Linux GB10 developer box and says so in `/proc/device-tree/model` — as
`NVIDIA_DGX_Spark`, where the underscores are separators, which is why the match
splits on them before applying word boundaries.

Picking it hands off with `plan="machine-setup"`, and the plan (not the task
text) is what swaps the build session's runbook: audit the box with the terminal
first — OS, updates, package manager, what's already installed, GPU and driver
on NVIDIA hardware — report what's there, propose a numbered plan, and only
install after an explicit `::ask`. Setting a machine up is the first task a new
user most wants and can least brief, so the agent does the briefing. It is also
the one job that needs no account anywhere, which is what the first build has
to be regardless.

The runbook opens with `machineDescription()`, and that line leads with age,
because age is what decides whether this job is real. On a machine unboxed this
week the drivers, updates and toolchain are genuinely undone and doing them is
worth an afternoon of someone's life; on a two-year-old machine most of it is
handled already, and an agent that doesn't know will "fix" things that were
never broken. The offer says the same thing out loud — it names the machine
as new, and names the work as the setup nobody enjoys — because being recognised
is what makes the offer land.

Also in the tree from the same lineage (dormant in the guided flow, used by the
login-mode dashboard flow): the generative first-screen system —
`FirstScreenCard` keep/drop picker, the living sketch pane
(first-screen-live.ts), populate pipeline (first-screen-populate.ts), and the
`context`/`first-screen`/`ready` directives. The guided runbook doesn't place
them; the components stay available for the build session's future artifacts.
