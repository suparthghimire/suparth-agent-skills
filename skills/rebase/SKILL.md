---
name: rebase
description: Rebase the current branch onto a remote branch (e.g. main), resolve conflicts, and report the result.
disable-model-invocation: true
argument-hint: <target branch>, <remote alias>
---

Sync the current branch with a branch on a remote. Usage: `/rebase <target>, <remote>`.

- 1st argument: `<target>`, the branch to rebase onto (e.g. `main`).
- 2nd argument: `<remote>`, the remote alias to pull from (e.g. `origin`, `upstream`).

Arguments may be separated by a comma, spaces, or both. `/rebase main, upstream` means: rebase the current branch onto `upstream/main`.

## 0. Preflight

### Arguments

If `<target>` or `<remote>` is missing, ask the user for whichever is missing before doing anything else. Don't assume `origin` or `main`. To help them choose, you may list the options with `git remote -v` (for the remote) or `git branch -r` (for the target).

### Confirm the setup

Before running any git command that changes anything (pull, rebase, stash, commit), look up the following with read-only commands and tell the user:

- **Current branch:** `git branch --show-current`
- **Target branch:** `<target>` (as `<remote>/<target>`)
- **Remote:** `<remote>` and its URL from `git remote get-url <remote>`

For example:

> Current branch: `feature/login`
> Target branch: `main` (`upstream/main`)
> Remote: `upstream` → `git@github.com:acme/app.git`

Stop and tell the user if `git remote get-url <remote>` fails (no remote with that name) or `<remote>/<target>` doesn't exist after `git fetch <remote> <target>`. Never pull from any remote other than `<remote>`.

### Checks

Run these and stop on any failure:

1. `<current>` is the current branch from above. If it is empty (detached HEAD) or equals `<target>`, stop and tell the user.
2. `git status --porcelain` must be empty. If not, show the changes and ask whether to commit, stash (`git stash push -u -m "rebase"`), or abort. If you stash, pop it at the end and report it.
3. If `.git/rebase-merge` or `.git/rebase-apply` exists, a rebase is already in progress. Ask whether to continue it or abort it before doing anything else.
4. `git rev-parse HEAD` gives `<start-sha>`. Keep it for the report and as a recovery point.

## 1. Pull the current branch from remote

```
git pull <remote> <current> -r
```

If `<remote>/<current>` doesn't exist (branch never pushed), skip this step and say so. If it stops with conflicts, resolve them as in step 3, then continue to step 2.

## 2. Pull the target branch

```
git pull <remote> <target> -r
```

This replays each local commit, one at a time, on top of `<remote>/<target>`. If it finishes cleanly, go to step 4.

## 3. Resolve conflicts

There are no exceptions: every conflict gets resolved. Never run `git rebase --abort`, `git rebase --skip`, `git checkout --ours/--theirs .`, or `git reset --hard` on your own to get out of a conflict. Only the user can decide to abort or drop a commit.

### What "ours" and "theirs" mean during a rebase

They are the reverse of a merge:

- **HEAD / "ours" / the upper side `<<<<<<< HEAD`** is the target branch (`<remote>/<target>`) plus the commits of this branch already replayed.
- **"theirs" / the lower side `>>>>>>> <sha> (<message>)`** is the single commit of this branch that is being replayed right now.

So a resolution must keep what the target branch changed **and** apply what this one commit meant to do on top of it.

### For each stopped commit

1. `git status` to list the conflicted files (`both modified`, `deleted by us`, `added by both`, ...).
2. See which commit is being replayed: `git log -1 REBASE_HEAD` and its full intent with `git show REBASE_HEAD`.
3. For each conflicted file, understand both sides before editing:
   - What this commit changed: `git show REBASE_HEAD -- <file>`
   - What the target changed since the branch forked: `git log --oneline <start-sha>..<remote>/<target> -- <file>`, then `git show <sha> -- <file>` for the relevant ones.
   - Read the whole conflicted region in the file, plus surrounding code it touches (callers, imports, types).
4. Edit the file to the correct combined result and remove every marker (`<<<<<<<`, `=======`, `>>>>>>>`). Typical cases:
   - Both sides added different things (imports, enum values, module providers, routes, DTO fields): keep both, dedupe, keep sorting if the file is sorted.
   - Target renamed or moved something this commit uses: apply this commit's change to the new name/location.
   - Target refactored code this commit also edited: re-apply this commit's intent onto the refactored version, don't restore the old code.
   - One side deleted a file the other modified: decide whether the modification is still needed somewhere else. If unsure, ask.
   - `pnpm-lock.yaml`: don't hand-merge. Take the target version (`git checkout --ours pnpm-lock.yaml` — "ours" is the target during a rebase), make sure the `package.json` files are resolved, then run `pnpm install` and `git add pnpm-lock.yaml`.
   - Prisma (`apps/*/prisma/schema.prisma`, `apps/*/prisma/migrations/`): merge the schema by keeping both sides' models/fields. Never edit or renumber a migration that exists on the target branch. If this branch adds a migration that conflicts in order or content with one on the target, ask the user.
   - Generated files (`dist`, build output, generated clients): take the target version and regenerate if needed.
5. **When you are not sure what to keep or remove, ask the user.** Show the file, both sides of the hunk, what each commit was trying to do, and your proposed resolution. Don't guess on business logic, behavior changes, or anything where both sides changed the same lines for different reasons.
6. Verify nothing is left: `git diff --check` and `grep -rn '^<<<<<<<\|^>>>>>>>' <files>` must return nothing.
7. `git add <files>`, then `GIT_EDITOR=true git rebase --continue` (avoids opening an editor for the commit message).
8. If the next commit stops again, repeat from 1. If git reports a commit became empty, it was already applied on the target: tell the user and ask before `git rebase --skip`.

## 4. Verify

After the rebase finishes:

- `git status` is clean and no rebase is in progress.
- `git log --oneline <remote>/<target>..HEAD` shows this branch's commits on top of the target.
- If any conflict was resolved, typecheck the affected apps/packages, e.g. `pnpm --filter <package> exec tsc --noEmit` (or `pnpm build --filter <package>`), and fix errors that come from the resolution. Run the tests of touched modules if they exist.
- If you stashed in step 0, `git stash pop` and report whether it applied cleanly.

Do not push. The branch history was rewritten, so pushing needs `git push --force-with-lease`; tell the user that and let them do it (or do it only if they ask).

## 5. Report

End with a short report:

- Current branch, target, remote (name and URL), `<start-sha>` → new `HEAD` sha.
- Result of step 1 and step 2 (up to date / fast-forwarded / rebased N commits).
- Each conflict: file, commit being replayed, what each side changed, how it was resolved, and whether the user decided it.
- Verification results (typecheck/tests) with any failures shown as-is.
- Anything left for the user: push with `--force-with-lease`, stash state, skipped commits.
- Recovery: `git reset --hard <start-sha>` restores the branch to how it was before the sync.
