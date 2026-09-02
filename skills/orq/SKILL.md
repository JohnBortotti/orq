---
name: orq
description: Run and coordinate other agents in tmux panes and git worktrees, and schedule recurring jobs, using the orq CLI. Use when work should be split across parallel agents, when you need to read or drive another agent's terminal, or when the user asks for something to happen on a schedule ("every day at 7pm", "every Friday morning").
---

# orq

`orq` is a thin CLI over `tmux`, `git worktree` and `systemd --user`. Each
subcommand is one or two calls to those tools with a decent name.

**This skill describes capability, not process.** It tells you what you can do
and where the sharp edges are. Who reviews whom, in what order, with what
roles — that belongs in the prompt you were given, never here.

Run `orq <command> --help` for exact flags. This document covers what `--help`
cannot: the model, the loop, and the things that will bite you.

---

## 1. The model

```
workspace          a tmux session, declared in ~/.orq/workspaces/<name>.yml
 └── worktree      a git worktree, one tmux window
      └── pane     a terminal
           └── agent   a pane WITH A NAME, in the registry
```

An **agent is a named pane**. A pane without a name is just a terminal; it
still exists, you just cannot address it by name (you can still act on it by
its `%id`).

Every workspace has a `_lead` window at its `root:` — one level above the
repos. That is where a workspace-level agent lives, which is usually you if
you are orchestrating. `orq spawn <name>` with no `--wt` puts an agent there.

**The registry is an index, not the truth.** The truth is git (for worktrees)
and systemd (for schedules). `~/.orq/state/` can be lost and `orq resurrect`
rebuilds what matters from `git worktree list`. Never write to it by hand.

Start here:

```bash
orq ls                 # every workspace: worktrees, panes, who is whose parent
orq ls --json          # the same, for parsing. { workspaces: [...], schedules: [...] }
```

`orq ls --json` is the contract. Anything orq knows comes out of it.

---

## 2. Doing work: worktrees and agents

```bash
orq wt new api/fc660 --branch feat/thing        # fresh worktree + window
orq wt new api/fc660 --branch feat/thing --spawn claude --prompt "..."
orq spawn reviewer --wt api/fc660 --prompt "review the diff"
orq spawn reviewer --wt api/fc660 --split       # a second agent in the same window
orq spawn lead                                   # no --wt: workspace level, in _lead
```

- The worktree is cut **fresh from `origin/<default>`**, not from the local
  branch. An existing branch is reused and never rebased.
- `--branch` is required and is **not yours to invent**. When the work comes
  from a ticket, the branch name comes from the ticket.
- The first agent in a window takes over the shell that is already there; the
  second onward splits.

Talking to an agent:

```bash
orq send reviewer "the diff is in HEAD~1, look at the error handling"
orq keys reviewer Escape                # named keys, separate from text on purpose
orq read reviewer --lines 80            # its screen, as plain text
orq kill reviewer                       # or: orq kill %31, for a pane with no name
```

Cleaning up:

```bash
orq wt rm api/fc660        # refuses if dirty, unpushed, or an agent is alive inside
orq resurrect              # windows died (reboot); rebuilds them from git
```

---

## 3. The loop: how you wait

There is **no `orq wait`**, and the absence is the design. A worker that
finishes calls `orq done`, which writes its status and sends a message to the
parent that spawned it.

You have two ways to wait, and both are first-class. Pick by whether you have
something else to do.

### End your turn (default)

Spawn, then **stop**. An agent that has ended its turn is idle waiting for
input, so the `done` message arrives as new input and wakes you.

```bash
orq spawn w1 --wt api/a --prompt "...  when finished run: orq done 'a is ready'"
orq spawn w2 --wt api/b --prompt "...  when finished run: orq done 'b is ready'"
# then end your turn. Say what you spawned and stop.
```

You wake **once per worker**, not when all of them finish. On each wake, run
`orq ls` to see who is still out.

### Poll with `orq read`

When you have work to do while waiting, or when the worker might die without
signalling, loop instead:

```bash
orq read w1 --lines 40      # look at its screen
orq ls                      # or check status, which the hooks keep current
```

Polling survives a worker that dies silently. Ending your turn does not — that
case is covered by the `Stop` hook, which marks the agent `done` even if it
forgot to call the command.

### Finishing, as a worker

```bash
orq done "PR 412 opened"          # to whoever spawned you
orq done --to lead "blocked on X" # to somebody else
```

`done` does **not** kill your pane. It means "I finished my turn", not "the
work is over" — the work ends at a merge, and cleanup belongs to `wt rm`. Your
pane stays up with the conversation on it, so a follow-up costs nothing.

---

## 4. Reading a screen

`orq read <name>` gives you the pane's visible screen. Nobody has enumerated
the states for you; read it and judge. A Claude Code pane shows things like
`✻ Churned for 1s · done 22:05` when it has finished, its input box when it is
waiting, and the whole trust dialog when it is stuck on one.

Two things will mislead you:

1. **A contextual suggestion is indistinguishable from typed text.** Claude
   Code shows greyed-out suggestions in its input box that look exactly like
   something somebody typed. Do not conclude a command was run because you see
   it on the prompt line.
2. **The screen is a moment, not a record.** It shows what fits, now.

Because of both, **status does not come from reading the screen**. Agents
report it themselves through a hook, and it is in `orq ls`:

- `working` — mid-turn
- `done` — turn ended
- (no status) — a pane with no provider, or one that has not run yet

⚠️ `done` means the *turn* ended. An agent that put a long command in the
background is `done` with work still running.

---

## 5. Sharp edges

**Names take no dots.** `^[a-z0-9][a-z0-9_/-]*$`. `:` and `.` are tmux target
separators, so a name with a dot becomes an ambiguous target. A repo called
`api.v2` gets a worktree called `api-v2`.

**`send` does not check whether the input is free.** It pastes and submits.
Claude Code queues what arrives mid-generation, so this is safe — but it means
`send` is never a reason to believe an agent was idle.

**`spawn --prompt` can refuse.** It waits for the agent to be ready to accept
input, and if it finds a blocking dialog instead it leaves the pane alive and
tells you. The prompt was not sent; resolve it on screen and use `orq send`.

**`wt rm` refuses for reasons worth reading**, not to be worked around:

| refusal | what it means |
|---|---|
| tree is dirty | uncommitted work, minus files orq itself put there |
| N commits not on origin | work that exists nowhere else |
| a live agent inside | somebody is working in it |
| a file you edited | orq put it there, you changed it — save it first |

**Do not reach for `--force`.** It exists for a human who has read the refusal
and decided. If you hit a refusal, report it; the refusal is information.

---

## 6. Schedules

A schedule is a systemd timer. Use one when the user wants something to happen
**on its own, repeatedly** — not to defer a single piece of work you could do
now.

### Translating what was asked

Your job is to turn what the user said into a calendar. `--cron` takes the
familiar five fields; `--calendar` takes systemd syntax directly when cron
cannot say it.

| what they said | what you write |
|---|---|
| every day at 7pm | `--cron "0 19 * * *"` |
| weekdays at 8am | `--cron "0 8 * * 1-5"` |
| every Friday at noon | `--cron "0 12 * * 5"` |
| every 30 minutes | `--cron "*/30 * * * *"` |
| the 1st of every month, 6:30am | `--cron "30 6 1 * *"` |
| every day at 7pm and 7am | two schedules, or `--calendar "*-*-* 07,19:00:00 <TZ>"` |

`orq schedule add` validates through `systemd-analyze` and prints the next
run. **Read it back to the user** — it is the cheapest way to catch a
misreading before it runs unattended for a week.

```bash
orq schedule add nightly --cron "3 19 * * *" \
  --desc "Daily error report, pre-triaged" \
  -- claude -p "summarise today's new errors, grouped by service"
```

- **`--desc` is not optional in spirit.** It is the sentence that shows up
  when somebody looks at the timer later and asks what it is for. Write what
  the user meant, not what the command does.
- **`--cwd`** is where the job runs. Default `~`. It sets a directory and
  nothing else — it does not tie the schedule to a workspace.
- Everything after `--` is the command, run through `bash -lc`. A schedule
  does not know what its command is; it only knows when.

Managing them:

```bash
orq schedule ls          # next run, last run, and whether the last one FAILED
orq schedule ls --json
orq schedule log nightly     # what the last run printed. The only way to see why it broke
orq schedule run nightly     # fire now, off schedule
orq schedule rm nightly
```

### What you must respect

- **The timezone is written into every unit, always.** When somebody says
  "7pm" they mean their 7pm, and the machine may be on UTC.
- **Whoever created it is recorded** — the unit, the log, and `schedule ls`
  all carry it. An agent creating a recurring job is a different kind of power
  from opening a visible pane: it runs later, alone, with nobody watching, and
  it can create more. The ugly case is not malice, it is accumulation.
- **There is a ceiling.** If you hit it, something has been accumulating; say
  so instead of finding room.
- **A schedule that fails every day looks healthy** in `systemctl`. `orq
  schedule ls` carries the last result for exactly this reason — check it
  before telling anybody a job is fine.

---

## 7. What this skill does not do

- **It does not define roles, order, or who reviews whom.** That is the
  prompt's job. This describes what the tools can do.
- **It does not invent branch names.** They come from the work — usually the
  ticket.
- **It does not write to `~/.orq/state/` or to unit files by hand.** Every
  change goes through a command, so the log and the state stay honest.
- **It does not use `--force`** on anything.

`orq dash` exists and is a dashboard for a human at a terminal. You do not
need it: everything it shows comes from `orq ls --json`.
