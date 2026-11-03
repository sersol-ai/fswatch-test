# Manual — two-way git sync on Windows

Operates the sync that keeps this folder (`C:\Users\Sergey\repos\fswatch-test`)
bidirectionally in sync with GitHub. Read this on the Windows side; every command
below is PowerShell, run from `C:\Users\Sergey\repos\fswatch-sync`.

**Nothing runs automatically until you turn it on** (step 5). Until then this
folder only changes when you run a cycle yourself.

---

## 1. How it behaves, in one paragraph

Local file edits are detected by fswatch and pushed to GitHub within a couple of
seconds. Changes pushed from anywhere else are pulled in on a timer. Both legs
run the same cycle. When your local copy and GitHub disagree about the same line,
**GitHub's version is kept and your version is discarded from the working tree** —
but never from git history, so it is always recoverable. Nothing here ever
force-pushes.

The cycle, in order:

```
git add            -> stage what changed (tracked-only, or everything: see config)
git commit         -> "sync: <UTC time>", only if something was staged
git fetch
git rebase         -> only if behind or diverged; conflicts resolved toward GitHub
git push           -> fast-forward only
```

---

## 2. Check what state it is in

```powershell
cd C:\Users\Sergey\repos\fswatch-sync
powershell -NoProfile -File .\sync.ps1 -Repo fswatch-test -Mode status
```

Reads local state only — no network, safe to run any time. Shows the local and
remote commits, how far ahead/behind you are, uncommitted files, whether sync is
halted, and the last six log lines.

```powershell
powershell -NoProfile -File .\install-task.ps1 -Repo fswatch-test -Action status
```

Adds: whether the watcher process is alive, how many fswatch processes exist, and
whether startup is registered.

---

## 3. Sync right now, by hand

```powershell
powershell -NoProfile -File .\sync.ps1 -Repo fswatch-test -Mode cycle
echo $LASTEXITCODE
```

| Exit | Meaning | What to do |
| --- | --- | --- |
| `0` | converged with GitHub | nothing |
| `1` | a git step failed (usually network) | it retries on the next cycle; check the log |
| `2` | **halted** — see step 6 | resolve, then resume |
| `3` | another cycle holds the lock | nothing, one is already running |

---

## 4. Do it deliberately: pause the automation

There is no pause switch. To work without the bot touching your edits, just don't
start the watcher, or stop it:

```powershell
Stop-Process -Name fswatch -Force          # stops event detection
Get-Process powershell | Where-Object { $_.Id -eq (Get-Content "$env:LOCALAPPDATA\fswatch-sync\fswatch-test.watch.pid") } | Stop-Process -Force
```

Anything you change while it is stopped is picked up the next time a cycle runs —
as a `sync:` commit. There is no way to stage a partial commit through this tool.

---

## 5. Turn it on

First, a pre-flight check. It parses the scripts and verifies every cmdlet
parameter and .NET member the tool calls really exists in your PowerShell, plus
that fswatch, git and the clone are where the config says. It executes nothing
and changes nothing:

```powershell
powershell -NoProfile -File .\check-syntax.ps1     # want: CLEAN, exit code 0
```

Foreground, both legs, stops with Ctrl-C:

```powershell
powershell -NoProfile -File .\watch.ps1 -Repo fswatch-test
```

Start at logon, no elevation required:

```powershell
powershell -NoProfile -File .\install-task.ps1 -Repo fswatch-test -Action runkey
powershell -NoProfile -File .\install-task.ps1 -Repo fswatch-test -Action status   # confirm
powershell -NoProfile -File .\install-task.ps1 -Repo fswatch-test -Action runkey-remove   # undo
```

Task Scheduler instead (survives logout, needs a normal interactive PowerShell
window — from a WSL shell `Register-ScheduledTask` fails with `0x80070005`):

```powershell
powershell -NoProfile -File .\install-task.ps1 -Repo fswatch-test -Action tasks
powershell -NoProfile -File .\install-task.ps1 -Repo fswatch-test -Action tasks-remove
```

One process carries both legs: fswatch for your local edits, a timer job for
everything everyone else pushed (a directory watch cannot see those).

---

## 6. When it halts

A halt means git could not merge and refused to guess — nothing was pushed, and
every later cycle refuses to run until you clear it. That is the safety valve.

```powershell
type "$env:LOCALAPPDATA\fswatch-sync\fswatch-test.halted"
```

Almost always you just need to accept GitHub's version and re-apply your work by
hand:

```powershell
cd C:\Users\Sergey\repos\fswatch-sync
powershell -NoProfile -File .\sync.ps1 -Repo fswatch-test -Mode resume
```

If your overwritten work was real, it is still in history — find it and take it
back:

```powershell
cd C:\Users\Sergey\repos\fswatch-test
git log --oneline -15
git show <commit> -- <path>
git checkout <commit> -- <path>
```

---

## 7. Reading the log

```powershell
gc "$env:LOCALAPPDATA\fswatch-sync\fswatch-test.log" -Tail 40
```

What the lines mean:

| Line | Meaning |
| --- | --- |
| `commit sync: 2026-11-03T10:47:37Z` | local changes were committed |
| `pushed 1 commit(s)` | they reached GitHub |
| `rebase onto refs/remotes/origin/main (behind=1 ahead=1)` | you had diverged; GitHub won |
| `OVERRIDDEN local work overridden by remote (1): path` | **your edit to that file was replaced by GitHub's** |
| `conflict resolved remote-wins, remote version kept: path` | modify/delete conflict settled toward GitHub |
| `HALT rebase could not be resolved` | halted, see step 6 |
| `ok in 1443ms` | a clean cycle. Suppressed for watcher/timer cycles, shown when you run one yourself |
| `watch: started pid=… pullJob=… interval=120s debounce=2s` | watcher is up, both legs armed |
| `watch: fswatch attached pid=…` | fswatch is running and its output is being read |
| `watch: fswatch stderr: …` | fswatch complaining. `Invalid handle when opening C:\C:\…` means a Windows path reached it instead of `/cygdrive/c/…` |
| `watch: fswatch stream ended, restart #1 in 5s` | fswatch exited; watcher is bringing it back (gives up after 10) |
| `watch: flushing pending cycle` | a change arrived just as fswatch stopped; the cycle ran before restarting it |
| `watch: stopped` | watcher exited |

Rotates to `.log.1` past 1 MB.

---

## 8. Configuration

`C:\Users\Sergey\repos\fswatch-sync\config.json`

| Key | Meaning |
| --- | --- |
| `debounceSec` | minimum gap between cycles; batches during a gap coalesce into one trailing cycle. Default 2 |
| `pullIntervalSec` | how often the inbound leg runs. Default 120 |
| `commitScope` | `all` = `git add -A` (any scratch file dropped in the folder becomes a permanent commit and is pushed). `tracked` = only files git already knows. Default `tracked`; this repo is set to `all` |
| `lockStaleSec` | a cycle older than this is treated as abandoned and its lock is taken over. Default 300 |

`.gitignore` is always honoured, so ignored files can never be committed. In
`infra` that means `secrets/` and `.env` are safe by construction.

**Note about this very file:** `commitScope` for this repo is `all`, so the next
cycle commits and pushes `MANUAL.md` to GitHub.

---

## 9. Adding another repo (`infra`)

Your WSL copy needs no changes. It keeps pushing to the same GitHub remote, so
WSL and Windows are two clones with GitHub as the referee — and on a real
conflict **whoever pushes first wins**, the other is rebased away.

```powershell
git clone https://github.com/sersol-ai/infra.git C:\Users\Sergey\repos\infra
git -C C:\Users\Sergey\repos\infra config --local core.autocrlf false
```

Add to `config.json` under `repos`:

```json
{ "id": "infra", "path": "C:\\Users\\Sergey\\repos\\infra", "commitScope": "tracked" }
```

Then `sync.ps1 -Repo infra -Mode cycle` / `watch.ps1 -Repo infra` /
`install-task.ps1 -Repo infra -Action runkey`. Use `commitScope: "tracked"` for
real repos so stray files never get committed.

---

## 10. Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| Windows edits never reach GitHub | watcher not running (`install-task.ps1 -Action status`); fswatch missing (`winget install --id emcrisostomo.fswatch -e`) |
| `watch: fswatch attached` appears but no cycles | check for `watch: fswatch stderr:` lines — that is fswatch rejecting the path. Then run `check-syntax.ps1` |
| `sync.ps1` exits at once saying git was not found | `config.json`'s `git` must be a full path (`C:\Program Files\Git\cmd\git.exe`), not `"git"` |
| GitHub changes never appear here | that is the **timer** leg, not fswatch — a watch can't see other people's pushes. Wait `pullIntervalSec`, or run a cycle |
| Sync does nothing at all | halted — step 6 |
| A file you edited got reverted | GitHub won. Check `OVERRIDDEN` in the log; recover from history, step 6 |
| `Invalid handle when opening C:\C:\Users…` | fswatch was given a Windows path. It needs `/cygdrive/c/...` — `watch.ps1` converts it; don't hand it a `C:\` path |
| Events take ~10 s instead of instant | fswatch fell back to its polling monitor. Re-check `fswatch --list-monitors` |
| Repeated `sync:` commits with no edits from you | line-ending churn: the clone needs `core.autocrlf false` and the repo needs `* text=auto` in `.gitattributes`. Already fixed in this repo |
| Commits full of `sync:` messages clutter history | expected; squash them when you merge, or use `commitScope: tracked` |
| Edit vanished within seconds of making it | it collided with a GitHub change inside the debounce window and remote-wins did its job. Don't author here without committing deliberately |

---

## 11. Uninstall

```powershell
cd C:\Users\Sergey\repos\fswatch-sync
powershell -NoProfile -File .\install-task.ps1 -Repo fswatch-test -Action runkey-remove
winget uninstall --id emcrisostomo.fswatch -e
```

Then delete the folder if you want. The clone itself is an ordinary git clone —
deleting it loses nothing that GitHub doesn't already have, except unpushed
`sync:` commits.
