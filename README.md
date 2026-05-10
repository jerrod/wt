# wt

A fast, opinionated CLI + TUI for managing per-feature [Claude Code](https://docs.claude.com/en/docs/claude-code) git worktrees.

It scans every `.claude/worktrees/<name>` directory under `$SITES_ROOT` (default `$HOME/Sites`), joins each one against:

- the repo's GitHub PRs (one cached `gh pr list` per repo)
- the live `tmux` session graph (which panes are in which worktree)
- the running process tree (is `claude` actually running?)
- rq plugin metadata (`.quality/session-prs.json`, `~/.claude/state/*/<branch>/rq/branch-issue.json`)

…and gives you one place to list, filter, preview, launch, prune, and bulk-restore those worktrees.

```
agent-plugins
● ancient-twirling-taco         worktree-ancient-twirling-taco            0m commit  18m edit  clean  PR#757 OPEN
● quiet-hopping-honey           issue-786-rq-gate-lint-shellcheck-…       4m commit  54m edit  clean  PR#788 merged
  inherited-tickling-bentley    worktree-inherited-tickling-bentley     22h commit  18h edit  clean  no-PR
  fix-519                       fix/519-humans-safe-scheduler            5d commit   5d edit  clean  PR#636 merged

arqu-web
  public-client                 fix/lucidrisq-web-public-client          2w commit   2w edit  DIRTY  PR#3471 merged
```

Green ● = a tmux pane lives in that worktree. Orange ● = `claude` is running there.

## Why

If you spin up many short-lived worktrees per issue/PR (the rq pattern), you end up with dozens of `.claude/worktrees/<silly-name>` directories scattered across your repos. Some are merged. Some are dirty scratch you forgot about. Some have a live tmux + claude session — others have stale ones. `git worktree list` does not tell you any of this. `wt` does.

## Install

```sh
git clone https://github.com/jerrod/wt.git ~/.local/share/wt
ln -s ~/.local/share/wt/wt ~/bin/wt          # or anywhere on your $PATH
```

Or one-line:

```sh
curl -fsSL https://raw.githubusercontent.com/jerrod/wt/main/wt -o ~/bin/wt && chmod +x ~/bin/wt
```

### Requirements

- `bash` 4+ (macOS: `brew install bash` — the system `/bin/bash` 3.2 will *not* work)
- `git`
- `gh` (authenticated; `gh auth status`)
- `tmux`
- `fzf` (only required for the interactive TUI; non-interactive subcommands work without it)
- `jq` (only required for the `--json` output and the claude-transcript section in previews)

### Configuration

`wt` is configured entirely via env vars — no config file:

| Var | Default | Purpose |
| --- | --- | --- |
| `SITES_ROOT` | `$HOME/Sites` | Where your repos live. `wt` scans `*/.claude/worktrees/*` under here. |
| `WT_GH_ORG` | `arqu-co` | GitHub org used for `gh pr list` lookups. |
| `WT_CACHE_DIR` | `$HOME/.cache/wt` | Per-repo PR cache directory. |
| `WT_CACHE_TTL` | `600` | Cache freshness in seconds. |
| `CLAUDE_BIN` | `claude` | The claude binary to launch. |

Override per invocation: `WT_GH_ORG=mycorp wt list`. Or export from your shell rc.

The launch flags are hard-coded to `--dangerously-skip-permissions --remote-control` — edit `LAUNCH_FLAGS` near the top of the script if you want different defaults.

## Usage

```
wt                                        # interactive TUI (multi-select + preview)
wt list                                   # grouped table, sorted most-recent-first per repo
wt list agent-plugins                     # substring match across repo|wt|branch
wt list --repo agent-plugins              # strict repo filter
wt list --claude                          # only rows with claude running
wt list --tmux                            # only rows with any tmux session
wt list --dirty                           # only DIRTY rows
wt list --open                            # only OPEN-PR rows
wt list --json                            # machine-readable (requires jq)
wt list --tsv                             # tab-separated, with header
wt all                                    # TUI including clean+no-PR rows
wt prune --dry-run                        # preview what would be removed
wt prune --yes                            # auto-remove merged + 7d-stale (skip dirty unless merged)
wt prune --force                          # also asks about non-merged dirty rows
wt restore --bound --dry-run              # preview which rq-bound worktrees would be reopened
wt restore --open-prs                     # open iTerm tabs for OPEN-PR worktrees not yet in tmux
wt restore                                # opens *every* not-live worktree (use with care)
wt 758                                    # auto-launch first match for "758"
wt -R list                                # refresh PR cache, then print
wt -h | --help                            # full help
```

### TUI

Run `wt`. The fzf picker shows one row per worktree with a 55%-width preview pane on the right.

| key | action |
| --- | --- |
| `enter` | launch/attach the focused row |
| `tab` | toggle multi-select |
| `ctrl-o` | open every selected worktree as a new iTerm tab (tmux + claude) |
| `ctrl-d` | prune every selected worktree (with confirm) |
| `ctrl-x` | kill the tmux session(s) for selected rows |
| `ctrl-r` | refresh the PR cache and re-render |
| `ctrl-/` | toggle the preview pane |

The preview pane shows: repo / worktree / branch / path, PR number+state+URL, live tmux panes (with claude marker), last 5 commits with relative ages, dirty file list, and the last few user/assistant turns from the most recent claude transcript JSONL.

### Filtering rules

The default view (used by `list`, `pick`, fzf TUI) hides rows that are clean + no-PR + no live session. `wt all` removes that default filter. Explicit filters (`--repo`, `--claude`, `--tmux`, `--dirty`, `--open`, positional substrings) compose with AND semantics.

### Status semantics

| Column | Meaning |
| --- | --- |
| `commit` age | `git log -1 --format=%ct` on the worktree's HEAD |
| `edit` age   | `stat -f %m` on the worktree's top-level dir |
| `DIRTY`      | `git diff --quiet` ≠ 0 OR untracked tracked-eligible files exist |
| `clean`      | working tree matches HEAD and no untracked files |
| `PR#NNN OPEN/MERGED/CLOSED` | from cached `gh pr list --head <branch>` |
| `no-PR`      | branch has no matching PR in the org |
| green ●      | tmux has at least one pane whose `pane_current_path` is the worktree (or under it) |
| orange ●     | a `claude` process is alive somewhere in that pane's process subtree |

### Pruning rules

`wt prune` walks every worktree and decides whether it is dead weight. A row is removable when any of the following hold:

- PR is `MERGED` (work landed; even dirty rows are removed because diffs are scratch by definition)
- PR is `CLOSED` and HEAD commit is older than 7 days
- there is no PR and HEAD commit is older than 7 days
- worktree is `clean` and has no PR and no live tmux/claude session

Rows with a live tmux/claude session are always skipped (`SKIP live`) so prune cannot yank a worktree out from under you. Rows that are `DIRTY` and *not* MERGED are skipped unless `--force`. `--dry-run` prints what would happen without touching anything. `--yes` skips per-row confirmation.

### Restoring after a reboot

`wt restore` is the morning-after command:

1. For every detached tmux session, opens an iTerm tab that runs `tmux attach -t <name>` (same behaviour as a standalone `tmux-reopen`).
2. For every worktree that is *not* currently in tmux, optionally filtered by `--bound` (rq metadata) or `--open-prs` (has an OPEN PR), spawns a fresh `tmux new-session -d` running claude, then opens an iTerm tab attached to it.

`--bound` is the safest "morning resume" filter — it only re-opens worktrees you were actively working on. `--dry-run` previews.

### Caching

`wt` collapses the per-branch PR lookups into a single `gh pr list --state all --limit 300 --json number,state,headRefName` call per repo, fanned out in parallel. Each repo's response is cached at `$WT_CACHE_DIR/<repo>.tsv` for `$WT_CACHE_TTL` seconds (default 10 minutes). Use `-R` / `--refresh` to invalidate, or `--no-cache` to bypass entirely.

Cold first run on ~50 worktrees across ~10 repos: ~6s. Warm: ~1.5s.

## Non-interactive use

Every interactive feature has a non-interactive counterpart. Examples:

```sh
# All dirty worktrees with no open PR, as paths, one per line:
wt list --json --dirty | jq -r '.[] | select(.pr == null) | .path'

# Count of OPEN PRs with claude actively running:
wt list --json --open --claude | jq 'length'

# Clean up everything safely:
wt prune --dry-run                        # see plan first
wt prune --yes                            # then commit
```

## Caveats

- The slug used to find Claude transcripts (`~/.claude/projects/<slug>`) follows the `path → -` substitution rule that Claude Code uses internally. If Anthropic changes the slug scheme, the transcript section in the preview will silently go blank. Everything else still works.
- The dirty rule is a heuristic. `wt` runs `git status --porcelain` per worktree but does not honor `.gitignore` overrides beyond what git itself reports.
- `wt restore` calls `osascript` to talk to iTerm. If you use a different terminal, edit the two `osascript` blocks (search for `tell application "iTerm"`).
- This is a personal tool. It assumes the rq plugin's worktree layout (`<repo>/.claude/worktrees/<name>`). It will not crash on other layouts but it also will not find them.

## License

MIT.
