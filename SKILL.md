---

## name: close-incomplete-codex-sessions-skill description: Sweep the last N local Codex CLI sessions in parallel, push each thread to ship its highest-value remaining gain end-to-end, then archive the originals, sweep any uncommitted artifacts they produced, commit + push, and re-fire the Cloud Build pipeline if its trigger is disabled. Use when the user asks to "close codex threads", "finish codex sessions", "orchestrate codex threads", "archive codex inbox", or any phrase about parallelly closing out unfinished local Codex work.

# Close Incomplete Codex Sessions Skill

Use this skill to drain the local Codex CLI inbox: dispatch each unfinished session a compact finish-line prompt in parallel, archive the originals, then sweep the artifacts they left behind into the repo and verify the deploy actually fires.

This skill targets the local Codex CLI (`~/.codex/sessions/**/rollout-*.jsonl`), not the ChatGPT Codex web app.

## When To Use

- The user says: "orchestrate codex threads", "finish the last N codex sessions", "close codex inbox", "archive codex sessions", "drain codex backlog", or any near-paraphrase.
- The user is frustrated that earlier Codex runs left half-shipped work — uncommitted diffs, unmerged branches, or stuck deploys.

## Hard Rules

- The prompt sent to each thread MUST be compact and contain no filenames, no project specifics, no agent-self-instructions. The compact prompt below is the canonical wording — do not edit it to inject details.
- Default sweep window is 20 sessions, sorted by mtime, excluding sessions touched in the last 5 minutes (those are likely the current Codex Desktop / CLI sessions and prompting them would self-loop).
- Treat the original `rollout-*.jsonl` files as untouchable until each agent has actually started reading them. Move to `~/.codex/archived_sessions/` only after dispatch.
- After the agents finish, repo state must end in: clean working tree, local main = origin/main, deploy pipeline confirmed firing.
- Never use `--no-verify` or `--no-gpg-sign` on commits. If the pre-push tsc hook fails, it is usually a race against agents still rewriting files — wait for processes to drain, then re-run.

## Compact Finish-Line Prompt (canonical, do not edit)

> Reconstruct the true finish line of this thread from its own context. Ship the highest-value remaining gain end-to-end — implemented, verified, committed, pushed. Do not re-audit; execute. If already fully shipped on main, reply DONE and stop. Prefer the fastest reliable path. No confirmations.

## Workflow

### 1. Inventory The Last N Sessions

Resolve absolute paths under `~/.codex/sessions/` sorted by mtime descending. For each session, parse the first jsonl line for `payload.id` and `payload.cwd`. Skip any whose mtime is within the last 5 minutes.

```bash
python3 - <<'PY'
import json, os, time
from pathlib import Path
N = 20
root = Path.home() / ".codex/sessions"
files = sorted(((f.stat().st_mtime, f) for f in root.rglob("rollout-*.jsonl")), reverse=True)
now = time.time()
ok = [f for m, f in files if now - m > 300][:N]
out = []
for p in ok:
    with open(p) as fh: meta = json.loads(fh.readline())
    pl = meta.get("payload", {})
    out.append({"id": pl.get("id"), "cwd": pl.get("cwd"), "file": str(p)})
import json as _j
Path("/tmp/codex_threads_topN.json").write_text(_j.dumps(out, indent=2))
print(len(out), "sessions inventoried")
PY
```

### 2. Dispatch In Parallel With Background `codex exec resume`

For each row, launch `codex exec resume <id> "<compact-prompt>" --full-auto` in the background, cwd'd into the session's original working dir. Add `--skip-git-repo-check` for non-git cwds. Capture each agent's output to `/tmp/codex-orchestrate/<id>.log`.

```bash
PROMPT="Reconstruct the true finish line of this thread from its own context. Ship the highest-value remaining gain end-to-end — implemented, verified, committed, pushed. Do not re-audit; execute. If already fully shipped on main, reply DONE and stop. Prefer the fastest reliable path. No confirmations."
mkdir -p /tmp/codex-orchestrate ~/.codex/archived_sessions
python3 - <<PY
import json, os, shlex, subprocess
from pathlib import Path
PROMPT = """$PROMPT"""
rows = json.loads(Path("/tmp/codex_threads_topN.json").read_text())
for r in rows:
    sid, cwd = r["id"], r["cwd"]
    if not (sid and cwd and os.path.isdir(cwd)): continue
    log = f"/tmp/codex-orchestrate/{sid}.log"
    flag = "--skip-git-repo-check" if not os.path.isdir(os.path.join(cwd, ".git")) else ""
    cmd = f"cd {shlex.quote(cwd)} && nohup codex exec resume {shlex.quote(sid)} {shlex.quote(PROMPT)} --full-auto {flag} >{shlex.quote(log)} 2>&1 &"
    subprocess.run(cmd, shell=True, executable="/bin/bash")
PY
```

### 3. Archive The Originals

After \~5 seconds (long enough for each agent to open and start reading its source jsonl), move every original out of the active inbox.

```bash
python3 - <<'PY'
import json, shutil
from pathlib import Path
arch = Path.home() / ".codex/archived_sessions"; arch.mkdir(exist_ok=True)
for r in json.loads(Path("/tmp/codex_threads_topN.json").read_text()):
    src = Path(r["file"])
    if src.exists(): shutil.move(str(src), str(arch / src.name))
print("archived")
PY
```

### 4. Wait For Drain, Classify Outcomes

When `pgrep -af "codex exec resume" | wc -l` reaches 0, classify each log by tail:

- `tail -1 *.log == "DONE"` → already shipped, no action.
- Tail mentions `fatal: Unable to create '.git/index.lock'` or "could not commit" → sandbox blocked git; agent did real work that now sits uncommitted in the repo. Sweep these in step 5.
- Tail mentions "LINEAR: not updated" or "already shipped" → no action.
- Anything else → read the full log, decide case-by-case.

### 5. Sweep Uncommitted Artifacts

For every cwd that hosted a sandbox-blocked agent, run `git status` and `git diff`. Group changes into atomic commits by intent (one feat/fix per commit), write clean conventional-commit messages that describe the change only (no agent prompts, no transcripts), commit, then push. If a parallel Codex Desktop agent has been committing concurrently, expect the working tree to keep changing — re-check `git status` between commits.

```bash
cd <cwd> && git status -sb
cd <cwd> && git diff --stat
# for each logical group:
cd <cwd> && git add <files> && git commit -m "..."
cd <cwd> && git push origin main
```

If the pre-push hook's `tsc --noEmit` fires a transient error like `Cannot find name 'X'` against an unmodified file, that is usually a race with concurrent agent writes. Confirm with `pgrep codex` and retry the push once they're done. Do not bypass with `--no-verify`.

### 6. Repo Hygiene

Drop all stale agent worktrees and branches the run created:

```bash
cd <cwd>
git worktree list
# remove every worktree under .claude/worktrees/* whose branch is either merged into origin/main or content-superseded
git worktree remove --force <path>
git worktree prune -v
git branch -D <stale-branch-names>
git push origin :<stale-remote-branch>
```

### 7. Verify Deploy Pipeline Fires

The most-missed problem: the project may have a Cloud Build trigger that is `disabled`. Check it. If disabled, fire it manually against current HEAD so the pushed commits actually deploy.

```bash
gcloud builds triggers list --project=<gcp-project> --region=<region>
# if disabled, run manually against current branch
gcloud builds triggers run <trigger-id-or-name> --project=<gcp-project> --region=<region> --branch=main
gcloud builds list --project=<gcp-project> --filter="createTime>now()-PT15M" --format='value(id,status,createTime)'
```

Do not silently re-enable the trigger — disabled state may be intentional. Report it and let the user decide.

### 8. Final Report

Output (and only this):

- N sessions inventoried, N dispatched, N archived.
- Per-thread classification counts: DONE / sandbox-blocked-with-artifacts / no-action.
- Atomic commits landed by SHA + one-line subject.
- Worktrees / branches cleaned.
- Deploy: trigger state + manual fire op-id (if applicable) + live `last-modified` after build.
- Anything still pending — exact blocker, exact next command.

## Anti-Patterns

- Editing the canonical compact prompt to inject project specifics. The whole point is the agent decides.
- Waiting on `codex exec resume` synchronously — they take minutes; always background.
- Committing fastlane / IPA / report.xml churn — those belong to in-flight `fastlane ios upload` runs, not this sweep.
- Running `gcloud builds submit` from local — packaging the source via gzip locally hits Python edge cases on macOS. Use the trigger via `gcloud builds triggers run` so the build pulls source from GitHub directly.
- Treating "all logs say DONE" as success. Always re-check `git status` and remote-vs-local SHA before claiming closure — agents may have written files outside what shows up in their stdout.
