---

## name: close-incomplete-codex-sessions-skill description: Sweep the last N local Codex CLI sessions in parallel, push each thread to ship its highest-value remaining gain end-to-end, archive the originals, sweep any uncommitted artifacts they produced, commit and push, and re-fire the Cloud Build pipeline if its trigger is disabled. Use when the user asks to close codex threads, finish codex sessions, orchestrate codex threads, archive codex inbox, drain codex backlog, or any phrase about parallelly closing out unfinished local Codex work.

# Close Incomplete Codex Sessions Skill

Drain the local Codex CLI inbox: send each unfinished session a compact finish-line prompt in parallel, archive the originals, sweep the artifacts the agents leave behind into the repo, then verify the deploy actually fires.

This skill targets the local Codex CLI sessions stored at `~/.codex/sessions/**/rollout-*.jsonl`, not the ChatGPT Codex web app.

## When To Use

- The user says: "orchestrate codex threads", "finish the last N codex sessions", "close codex inbox", "archive codex sessions", "drain codex backlog", or any near-paraphrase.
- The user is frustrated that earlier Codex runs left half-shipped work, uncommitted diffs, unmerged branches, or stuck deploys.

## Hard Rules

- The prompt sent to each thread must be compact and contain no filenames, no project specifics, no agent-self-instructions. The compact prompt below is canonical, do not edit it to inject details.
- Default sweep window is 20 sessions sorted by mtime, excluding sessions touched in the last 5 minutes. Those are likely the current Codex Desktop or CLI sessions and prompting them would self-loop.
- Treat the original `rollout-*.jsonl` files as untouchable until each agent has actually started reading them. Move to `~/.codex/archived_sessions/` only after dispatch.
- After the agents finish, repo state must end in: clean working tree, local main equals origin/main, deploy pipeline confirmed firing.
- Never use `--no-verify` or `--no-gpg-sign`. If the pre-push tsc hook fails with a phantom error, it is usually a race against agents still rewriting files. Wait for processes to drain, then re-run.

## Compact Finish-Line Prompt (canonical, do not edit)

> Reconstruct the true finish line of this thread from its own context. Ship the highest-value remaining gain end-to-end, implemented, verified, committed, pushed. Do not re-audit, execute. If already fully shipped on main, reply DONE and stop. Prefer the fastest reliable path. No confirmations.

## Workflow

### 1. Inventory The Last N Sessions

Resolve absolute paths under `~/.codex/sessions/` sorted by mtime descending. For each, parse the first jsonl line for `payload.id` and `payload.cwd`. Skip any whose mtime is within the last 5 minutes.

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
Path("/tmp/codex_threads_topN.json").write_text(json.dumps(out, indent=2))
print(len(out), "sessions inventoried")
PY
```

### 2. Dispatch In Parallel

For each row, launch `codex exec resume <id> "<compact-prompt>" --full-auto` in the background, cwd into the session's original working dir. Add `--skip-git-repo-check` for non-git cwds. Stream each agent's output to `/tmp/codex-orchestrate/<id>.log`.

```bash
PROMPT="Reconstruct the true finish line of this thread from its own context. Ship the highest-value remaining gain end-to-end, implemented, verified, committed, pushed. Do not re-audit, execute. If already fully shipped on main, reply DONE and stop. Prefer the fastest reliable path. No confirmations."
mkdir -p /tmp/codex-orchestrate ~/.codex/archived_sessions
PROMPT="$PROMPT" python3 - <<'PY'
import json, os, shlex, subprocess
from pathlib import Path
PROMPT = os.environ["PROMPT"]
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

After about 5 seconds, move every original out of the active inbox.

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

When `pgrep -af "codex exec resume" | wc -l` reaches 0, classify each log by its tail:

- Tail equals `DONE` -&gt; already shipped, no action.
- Tail mentions `fatal: Unable to create '.git/index.lock'` or `could not commit` -&gt; sandbox blocked git, the agent did real work that now sits uncommitted in the repo. Sweep these in step 5.
- Tail mentions `LINEAR: not updated` or `already shipped` -&gt; no action.
- Anything else -&gt; read the full log, decide case-by-case.

### 5. Sweep Uncommitted Artifacts

For every cwd that hosted a sandbox-blocked agent, run `git status` and `git diff`. Group changes into atomic commits by intent, write clean conventional-commit messages that describe the change only, no agent prompts, no transcripts. Commit, then push. If a parallel Codex Desktop agent is committing concurrently, expect the working tree to keep changing. Re-check `git status` between commits.

If the pre-push hook's `tsc --noEmit` fires a transient error like `Cannot find name X` against an unmodified file, that is usually a race with concurrent agent writes. Confirm with `pgrep codex` and retry once they are done. Do not bypass with `--no-verify`.

### 6. Repo Hygiene

Drop stale agent worktrees and branches the run created.

```bash
git worktree list
git worktree remove --force <path>
git worktree prune -v
git branch -D <stale-branch-names>
git push origin :<stale-remote-branch>
```

Only drop a branch when its sha is either reachable from `origin/main` or content-superseded by a newer commit on main.

### 7. Verify Deploy Pipeline Fires

The most-missed problem: the project may have a Cloud Build trigger that is disabled. Check it. If disabled, fire it manually against current HEAD so the pushed commits actually deploy.

```bash
gcloud builds triggers list --project=<gcp-project> --region=<region>
gcloud builds triggers run <trigger-id-or-name> --project=<gcp-project> --region=<region> --branch=main
gcloud builds list --project=<gcp-project> --filter="createTime>now()-PT15M" --format='value(id,status,createTime)'
```

Do not silently re-enable the trigger, disabled state may be intentional. Report it and let the user decide.

### 8. Final Report

Output and only this:

- N inventoried, N dispatched, N archived.
- Per-thread classification counts: DONE, sandbox-blocked-with-artifacts, no-action.
- Atomic commits landed by sha plus one-line subject.
- Worktrees and branches cleaned.
- Deploy: trigger state, manual-fire op-id if applicable, live `last-modified` after build.
- Anything still pending: exact blocker, exact next command.

## Anti-Patterns

- Editing the canonical compact prompt to inject project specifics. The whole point is that the agent decides.
- Waiting on `codex exec resume` synchronously. They take minutes, always background.
- Committing fastlane, IPA, or report.xml churn. Those belong to in-flight `fastlane ios upload` runs, not this sweep.
- Running `gcloud builds submit` from local. Packaging the source via gzip locally hits Python edge cases on macOS. Use the trigger via `gcloud builds triggers run` so the build pulls source from GitHub directly.
- Treating "all logs say DONE" as success. Always re-check `git status` and remote-vs-local sha before claiming closure, agents may have written files outside their stdout.
