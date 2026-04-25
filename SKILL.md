---
name: close-incomplete-codex-sessions-skill
description: Sweep the last N local Codex CLI sessions in parallel, push each thread to ship its highest-value remaining gain end-to-end, archive the originals, sweep any uncommitted artifacts they produced, commit and push, and re-fire the Cloud Build pipeline if its trigger is disabled. Use when the user asks to close codex threads, finish codex sessions, orchestrate codex threads, archive codex inbox, drain codex backlog, or any phrase about parallelly closing out unfinished local Codex work.
---

# Close Incomplete Codex Sessions Skill

Drain the local Codex CLI inbox: send each unfinished session a compact finish-line prompt in parallel, archive the originals, sweep the artifacts the agents leave behind into the repo, then verify the deploy actually fires.

Targets the local Codex CLI sessions stored at ~/.codex/sessions/**/rollout-*.jsonl, not the ChatGPT Codex web app.

## When To Use

- The user says: "orchestrate codex threads", "finish the last N codex sessions", "close codex inbox", "archive codex sessions", "drain codex backlog", or any near-paraphrase.
- The user is frustrated that earlier Codex runs left half-shipped work, uncommitted diffs, unmerged branches, or stuck deploys.

## Hard Rules

- The prompt sent to each thread must be compact and contain no filenames, no project specifics, no agent-self-instructions. The compact prompt below is canonical, do not edit it to inject details.
- Default sweep window is 20 sessions sorted by mtime, excluding sessions touched in the last 5 minutes. Those are likely the current Codex Desktop or CLI sessions and prompting them would self-loop.
- Treat the original rollout-*.jsonl files as untouchable until each agent has actually started reading them. Move to ~/.codex/archived_sessions/ only after dispatch.
- After the agents finish, repo state must end in: clean working tree, local main equals origin/main, deploy pipeline confirmed firing.
- Never use --no-verify or --no-gpg-sign. If the pre-push tsc hook fails with a phantom error, it is usually a race against agents still rewriting files. Wait for processes to drain, then re-run.

## Compact Finish-Line Prompt (canonical, do not edit)

> Reconstruct the true finish line of this thread from its own context. Ship the highest-value remaining gain end-to-end, implemented, verified, committed, pushed. Do not re-audit, execute. If already fully shipped on main, reply DONE and stop. Prefer the fastest reliable path. No confirmations.

## Workflow

### 1. Inventory The Last N Sessions

Resolve absolute paths under ~/.codex/sessions/ sorted by mtime descending. For each, parse the first jsonl line for payload.id and payload.cwd. Skip any whose mtime is within the last 5 minutes.

### 2. Dispatch In Parallel

For each row, launch codex exec resume <id> "<compact-prompt>" --full-auto in the background, cwd into the session's original working dir. Add --skip-git-repo-check for non-git cwds. Stream each agent's output to /tmp/codex-orchestrate/<id>.log.

### 3. Archive The Originals

After about 5 seconds, move every original out of the active inbox into ~/.codex/archived_sessions/.

### 4. Wait For Drain, Classify Outcomes

When pgrep -af codex exec resume reaches 0, classify each log by its tail.

- Tail equals DONE: already shipped, no action.
- Tail mentions fatal: Unable to create .git/index.lock or could not commit: sandbox blocked git, the agent did real work that now sits uncommitted in the repo.
- Tail mentions LINEAR: not updated or already shipped: no action.
- Anything else: read the full log, decide case-by-case.

### 5. Sweep Uncommitted Artifacts

For every cwd that hosted a sandbox-blocked agent, run git status and git diff. Group changes into atomic commits by intent, write clean conventional-commit messages that describe the change only, no agent prompts, no transcripts. Commit, then push. If a parallel Codex Desktop agent is committing concurrently, expect the working tree to keep changing. Re-check git status between commits.

If the pre-push hook tsc --noEmit fires a transient error like Cannot find name X against an unmodified file, that is usually a race with concurrent agent writes. Confirm with pgrep codex and retry once they are done. Do not bypass with --no-verify.

### 6. Repo Hygiene

Drop stale agent worktrees and branches the run created. Use git worktree list, git worktree remove --force, git worktree prune -v, git branch -D, git push origin :branch. Only drop a branch when its sha is reachable from origin/main or content-superseded by a newer commit on main.

### 7. Verify Deploy Pipeline Fires

The most-missed problem: the project may have a Cloud Build trigger that is disabled. Check it with gcloud builds triggers list. If disabled, fire it manually against current HEAD with gcloud builds triggers run so the pushed commits actually deploy. Do not silently re-enable, disabled state may be intentional. Report it and let the user decide.

### 8. Final Report

Output and only this: N inventoried/dispatched/archived, per-thread classification counts, atomic commits landed by sha plus one-line subject, worktrees and branches cleaned, deploy trigger state and manual-fire op-id and live last-modified after build, anything still pending with exact blocker and exact next command.

## Anti-Patterns

- Editing the canonical compact prompt to inject project specifics. The whole point is that the agent decides.
- Waiting on codex exec resume synchronously. They take minutes, always background.
- Committing fastlane, IPA, or report.xml churn. Those belong to in-flight fastlane ios upload runs, not this sweep.
- Running gcloud builds submit from local. Packaging the source via gzip locally hits Python edge cases on macOS. Use gcloud builds triggers run so the build pulls source from GitHub directly.
- Treating "all logs say DONE" as success. Always re-check git status and remote-vs-local sha before claiming closure, agents may have written files outside their stdout.
