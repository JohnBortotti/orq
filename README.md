# orq

Agent orchestrator over `tmux`, `git worktree` and `systemd`.

A thin CLI: each subcommand is one or two calls to tmux, git or systemd with a
decent name. The orchestration -- the graph, the order, who does what -- does
not live here; it lives in the skill and in the prompt.

**The plan is the source of truth**, and it lives outside this repo:
`notes/projects/orq/index.md` in the vault.

## Layout

| file | what it is |
|---|---|
| `orq` | the CLI. One file, stdlib + PyYAML. No build, readable in an afternoon, fixable at 2am. |
| `orq-dash` | the dashboard. A separate program that talks to `orq` only through `ls --json`, `schedule ls --json` and `read`. Needs the venv. |

`orq dash` runs `orq-dash` for you. It is deliberately the ONLY way in:
`orq-dash` is not put on PATH, because one command surface is the point --
`orq` finds it as a sibling file, and never imports Textual, so the
orchestrator stays dependency-light.

## Install

```bash
ln -s ~/workspace/orq/orq ~/.local/bin/orq   # only this one goes on PATH

# the dashboard only
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
```

Requires `tmux`, `git`, `systemd --user` and `python3`.
