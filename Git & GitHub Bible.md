# Git & GitHub Bible

Reference for every Git and GitHub workflow Joe runs. Cookbook-first. Read the **Safety Rules** section once. After that, jump straight to the scenario you're in.

All examples assume you are in the repo:
```bash
cd ~/Documents/Development/<repo>
```
Substitute your actual repo path. Worktree paths look like `~/Documents/Development/<repo>/.claude/worktrees/<name>` and the same commands apply inside them.

Cross-references: [[Multi-Instance Protocol]], [[Claude Code Setup]], [[dev-playbook]].

---

## 1. Safety Rules

| ⚠ Rule | Why |
|---|---|
| **Never** `git push --force` to `main` / `master` / shared branches | Rewrites history other people depend on. Use `--force-with-lease` only on **your own** feature branches. |
| **Never** `git reset --hard` without first running `git status` and `git stash` if there's anything uncommitted | `--hard` discards working-dir changes silently. No undo unless they were committed or stashed. |
| **Never** `git rebase` a branch that's already been merged into `main` or pulled by someone else | Rewrites SHAs; collaborators (or other machines) will diverge. |
| **Never** commit secrets (`.env`, keys, tokens). If you do, **rotate the secret** first, then scrub history | History rewrite alone doesn't help — the secret is already exposed. |
| **Always** `git fetch` before assuming you know the remote state | Local view of `origin/main` is only as fresh as your last fetch. |
| **Always** create a new commit instead of `--amend` after a push | Amend rewrites the last commit. If it's pushed, others may have it. |
| **Always** prefer `--force-with-lease` over `--force` when force-push is unavoidable | `--force-with-lease` aborts if the remote moved since your last fetch. `--force` blindly overwrites. |
| **Never** skip pre-commit hooks (`--no-verify`) without a documented reason | Hooks exist to catch real problems; bypassing them shifts the bug to CI or prod. |

If you find yourself reaching for any ⚠ command, pause and read the matching scenario in §6 first.

---

## 2. Mental Model

Git tracks state across **three trees**:

```
Working Directory   ←  files you can see and edit
        ↓ git add
Index (Staging)     ←  what will be in your next commit
        ↓ git commit
HEAD (Committed)    ←  the current commit on your current branch
        ↓ git push
Remote              ←  origin/<branch> on GitHub
```

| Concept | What it is |
|---|---|
| **Commit** | An immutable snapshot of the index, identified by a SHA-1 hash. |
| **Branch** | A movable pointer to a commit. `main`, `feature/x`, etc. |
| **HEAD** | A pointer to the commit you currently have checked out. Usually points at a branch (which points at a commit). |
| **Ref** | Any name that points at a commit: a branch, a tag, `HEAD`, `HEAD~3`, `HEAD@{2}`. |
| **Remote** | A named URL to another repo. `origin` is the default. `upstream` is the convention for the repo you forked from. |
| **Upstream branch** | The remote branch your local branch tracks. Set with `git push -u origin <branch>` or `git branch --set-upstream-to`. |
| **Fast-forward** | A merge where the target branch's tip is a direct ancestor of the source. No merge commit needed — Git just moves the pointer. |

**Key insight:** branches are cheap pointers, not directories. `git switch other-branch` doesn't copy files — it moves HEAD and updates your working dir. This is why operations like `cherry-pick` and `rebase` can shuffle commits between branches so easily.

---

## 3. Daily Golden Path

The 90% workflow:

| Step | Command | Notes |
|---|---|---|
| 1. Sync main | `git switch main && git pull` | Always start from current main |
| 2. Branch | `git switch -c feat/short-name` | One branch per logical change |
| 3. Edit | (work) | Small, focused commits |
| 4. Stage | `git add <file>` or `git add -p` | Prefer named files over `git add .` |
| 5. Commit | `git commit -m "feat: short imperative summary"` | First line ≤72 chars |
| 6. Push | `git push -u origin feat/short-name` | `-u` only the first time |
| 7. Open PR | `gh pr create --title "..." --body "..."` | See §9 for the full heredoc body pattern (Summary + Test plan) |
| 8. Iterate | edit → `git add` → `git commit` → `git push` | No `-u` after first push |
| 9. Merge | `gh pr merge --squash --delete-branch` | Or wait for review → merge in UI |
| 10. Clean up | `git switch main && git pull && git branch -d feat/short-name` | `-d` (lowercase) refuses if unmerged; safe |

**Commit message style** (no team standard enforced; this is a reasonable default):
- Subject line: `<type>: <imperative summary>` where type ∈ {feat, fix, refactor, docs, test, chore}
- Body (optional): why, not what. The diff shows what.

---

## 4. Decision Matrices

### 4.1 Merge vs Rebase

| Situation | Use |
|---|---|
| Integrating a long-lived feature into main, want to preserve branch history | `git merge --no-ff` |
| Updating your in-flight feature branch with latest main | `git rebase main` (cleaner history) **or** `git merge main` (safer if branch is shared) |
| Branch is shared with another machine or another person | `git merge` — never rebase shared history |
| Branch exists locally on multiple of your own machines but nothing's been pulled from origin yet | Rebase OK — "shared" means *pulled by someone else*, not just *pushed* |
| Cleaning up your local commits before opening a PR | `git rebase -i` to squash/reorder |
| You want a single commit on main per feature | Use the **PR's "Squash and merge"** button (or `gh pr merge --squash`) |

**Default for solo feature branches:** rebase locally for cleanup, then `gh pr merge --squash` for the final integration.

### 4.2 Reset Modes

| Mode | HEAD moves? | Index reset? | Working dir reset? | When to use |
|---|---|---|---|---|
| `--soft` | yes | no | no | Undo a commit but keep all changes staged. Common for "I want to redo my last commit." |
| `--mixed` (default) | yes | yes | no | Undo a commit, unstage changes, but keep edits in working dir. |
| `--hard` ⚠ | yes | yes | yes | Throw everything away back to the target ref. **Destructive — no undo unless reflog catches it.** |

`HEAD~1` = one commit back. `HEAD~3` = three commits back. `HEAD@{2}` = where HEAD was 2 movements ago (per reflog).

### 4.3 Reset vs Revert vs Checkout

| Goal | Command |
|---|---|
| Undo a commit on a **private** branch (rewrite history) | `git reset` |
| Undo a commit on a **shared/pushed** branch (preserve history) | `git revert` (creates new commit that inverts) |
| Discard uncommitted changes to one file | `git restore <file>` |
| Discard all uncommitted changes | `git restore .` ⚠ (or `git stash` if you might want them later) |
| Switch to a different branch | `git switch <branch>` |
| Switch to a specific commit (detached HEAD, read-only) | `git switch --detach <sha>` |

`git checkout` still works for all of these but `switch` and `restore` (Git ≥2.23) are clearer — `checkout` was overloaded.

### 4.4 Force-Push Decision Tree

```
Need to push but remote rejects?
├─ Did you rebase or amend?
│   ├─ Yes, branch is YOURS only          → git push --force-with-lease ✓
│   ├─ Yes, branch is shared              → STOP. Talk to collaborators first.
│   └─ Yes, branch is main/master         → STOP. Never force-push main.
└─ No, just behind                        → git pull --rebase, then git push
```

Always prefer `--force-with-lease` over `--force`. It refuses to push if the remote has commits you haven't fetched, preventing you from clobbering someone else's work.

---

## 5. Command Reference

### 5.1 Inspecting State

| Command | Purpose |
|---|---|
| `git status` | What's staged, modified, untracked. **First command of every session.** |
| `git status -s` | Short format |
| `git log --oneline -20` | Last 20 commits, one per line |
| `git log --graph --oneline --all -20` | Visual branch graph |
| `git log -p <file>` | Every commit that touched a file, with diffs |
| `git log -S "string"` | "Pickaxe" — every commit that added/removed that string |
| `git log -G "regex"` | Pickaxe with regex |
| `git diff` | Unstaged changes |
| `git diff --staged` | Staged but uncommitted changes |
| `git diff main..HEAD` | Everything different between main and current branch |
| `git show <sha>` | The diff and message of one commit |
| `git blame <file>` | Per-line, who last changed it and in which commit |
| `git reflog` | Every HEAD movement (the "undo journal") |

### 5.2 Staging & Committing

| Command | Purpose |
|---|---|
| `git add <file>` | Stage a specific file |
| `git add -p` | Stage hunks interactively (review every change before staging) |
| `git add .` | Stage everything in current dir ⚠ — risk of staging secrets/junk |
| `git restore --staged <file>` | Unstage a file (keeps edits) |
| `git commit -m "msg"` | Commit staged changes |
| `git commit -a -m "msg"` | Stage all tracked-file changes and commit (skips untracked) |
| `git commit --amend` | Replace the last commit (message + staged changes) ⚠ if pushed |
| `git commit --amend --no-edit` | Add staged changes to last commit, keep message |

### 5.3 Branching

| Command | Purpose |
|---|---|
| `git branch` | List local branches |
| `git branch -a` | List local + remote-tracking branches |
| `git branch -vv` | List with upstream tracking info |
| `git switch <branch>` | Switch to existing branch |
| `git switch -c <new>` | Create + switch to new branch from current HEAD |
| `git switch -c <new> origin/<remote-branch>` | Create local branch tracking a remote |
| `git branch -d <branch>` | Delete branch (refuses if unmerged) |
| `git branch -D <branch>` ⚠ | Force-delete (loses commits if unmerged) |
| `git branch -m <old> <new>` | Rename branch |

### 5.4 Syncing

| Command | Purpose |
|---|---|
| `git fetch` | Download remote refs/objects, don't merge. **Safe to run anytime.** |
| `git fetch --all --prune` | Fetch all remotes; remove tracking refs for branches deleted on remote |
| `git pull` | `fetch` + `merge` (default) on the current branch |
| `git pull --rebase` | `fetch` + `rebase` instead of merge — cleaner history when collaborating |
| `git push` | Push current branch to its upstream |
| `git push -u origin <branch>` | First push: set upstream tracking |
| `git push --force-with-lease` | Force-push with safety check (see §4.4) |
| `git push --tags` | Push annotated tags too (not pushed by default) |

**Set `pull.rebase = true` globally** if you want pulls to always rebase (avoids "Merge branch 'main' of github..." commits):
```bash
git config --global pull.rebase true
```

### 5.5 Integrating

| Command | Purpose |
|---|---|
| `git merge <branch>` | Merge `<branch>` into current. Fast-forward if possible, else merge commit. |
| `git merge --no-ff <branch>` | Always create a merge commit (preserves branch shape in history) |
| `git merge --squash <branch>` | Squash all of `<branch>` into staged changes on current; you commit manually |
| `git merge --abort` | Bail out of an in-progress merge |
| `git rebase <base>` | Replay current branch's commits on top of `<base>` |
| `git rebase -i <base>` | Interactive: pick/squash/reword/edit/drop each commit |
| `git rebase --onto <new-base> <old-base> <branch>` | Surgical: move `<branch>` from old-base to new-base |
| `git rebase --continue` | After resolving conflicts, continue the rebase |
| `git rebase --skip` | Skip the current commit being rebased (rare) |
| `git rebase --abort` | Bail out of an in-progress rebase, return to pre-rebase state |

### 5.6 Moving Commits

| Command | Purpose |
|---|---|
| `git cherry-pick <sha>` | Apply that commit to current branch (creates new SHA) |
| `git cherry-pick <sha1>..<sha2>` | Pick a range (exclusive of sha1) |
| `git cherry-pick -x <sha>` | Add "(cherry picked from commit ...)" to message |
| `git cherry-pick --abort` | Bail out if conflicted |
| `git cherry-pick --continue` | After resolving conflicts |
| `git revert <sha>` | Create a new commit that undoes `<sha>`. Safe on shared history. |
| `git revert -n <sha>` | Stage the revert without committing (combine multiple reverts) |
| `git revert <sha1>..<sha2>` | Revert a range |

### 5.7 Saving Work In Progress

| Command | Purpose |
|---|---|
| `git stash` | Save unstaged + staged changes; revert working dir to HEAD |
| `git stash push -m "msg"` | Stash with a label |
| `git stash push -u` | Include untracked files (`-u` = `--include-untracked`) |
| `git stash list` | Show all stashes |
| `git stash show -p 'stash@{0}'` | Show diff of a stash |
| `git stash pop` | Apply latest stash and drop it |
| `git stash pop 'stash@{2}'` | Apply specific stash and drop it |
| `git stash apply 'stash@{0}'` | Apply but **don't** drop (safer for risky restores) |
| `git stash drop 'stash@{0}'` | Delete a specific stash |
| `git stash clear` ⚠ | Delete **all** stashes |

**zsh quoting:** wrap any `stash@{N}` or `HEAD@{N}` argument in single quotes. zsh interprets `{...}` as brace expansion and the bare form errors. Bash users can drop the quotes if preferred.

### 5.8 Recovery (Reflog & Friends)

| Command | Purpose |
|---|---|
| `git reflog` | Every HEAD movement in this repo, newest first. **Your time machine.** |
| `git reflog show <branch>` | Reflog for one branch |
| `git reset --hard 'HEAD@{2}'` ⚠ | Jump HEAD back to where it was 2 movements ago (quote braces in zsh) |
| `git reset --hard <sha>` ⚠ | Jump HEAD to a specific commit |
| `git fsck --lost-found` | Find dangling commits (orphaned by rebases/resets) |
| `git cat-file -p <sha>` | Inspect any object by SHA |

Reflog entries persist for ~90 days by default (`gc.reflogExpire`). If you "lose" a commit via reset/rebase, it's almost always still in the reflog.

### 5.9 Investigation

| Command | Purpose |
|---|---|
| `git bisect start` | Begin a binary-search bug hunt |
| `git bisect bad` | Mark current commit as bad |
| `git bisect good <sha>` | Mark a known-good commit |
| `git bisect reset` | Exit bisect, return to original HEAD |
| `git bisect run <script>` | Automated bisect — `<script>` exits 0 for good, non-zero for bad |
| `git blame -L 10,20 <file>` | Blame lines 10–20 only |
| `git log --follow <file>` | Log including history before file was renamed |

### 5.10 Tags & Releases

| Command | Purpose |
|---|---|
| `git tag` | List all tags |
| `git tag -a v1.2.0 -m "Release 1.2.0"` | Create annotated tag (recommended for releases) |
| `git tag v1.2.0` | Create lightweight tag (no metadata; for personal markers) |
| `git tag -d v1.2.0` | Delete local tag |
| `git push origin v1.2.0` | Push one tag |
| `git push --tags` | Push all annotated tags |
| `git push --delete origin v1.2.0` | Delete remote tag |
| `gh release create v1.2.0 --notes "..."` | Create a GitHub Release attached to the tag |
| `gh release create v1.2.0 dist/*.zip` | Release with binary assets attached |

### 5.11 Worktrees

A worktree is a separate working directory backed by the same `.git` repo. Lets you have `main` checked out in one folder and `feat/x` in another simultaneously. Claude Code uses these per-task.

| Command | Purpose |
|---|---|
| `git worktree list` | Show all worktrees on this repo |
| `git worktree add .claude/worktrees/feat-x feat/x` | Create worktree at that path checked out to `feat/x` (Joe's convention; see §7) |
| `git worktree add -b feat/y .claude/worktrees/feat-y main` | Create worktree + new branch off main |
| `git worktree remove <path>` | Remove worktree (refuses if dirty) |
| `git worktree remove --force <path>` ⚠ | Force-remove dirty worktree |
| `git worktree prune` | Clean up admin records for worktrees whose dirs are gone |

See §7 for Joe-specific worktree workflows.

### 5.12 Submodules (use sparingly)

| Command | Purpose |
|---|---|
| `git submodule add <url> <path>` | Add a submodule |
| `git submodule update --init --recursive` | Initialize + check out all submodules after a fresh clone |
| `git clone --recurse-submodules <url>` | Clone with submodules in one shot |

**Avoid submodules unless absolutely necessary.** They complicate every workflow. Prefer monorepo or package managers.

---

## 6. Scenario Cookbook

Format: **Situation → What's happening → Run this → Verify → Gotchas.**

### 6.1 I committed to the wrong branch

**What's happening:** You meant to be on `feat/x` but you were on `main` when you committed.

**Run this** (commit not yet pushed):
```bash
git log -1                            # note the SHA of the commit to move
git log --oneline origin/main..HEAD   # ⚠ MUST show exactly one commit — if more, there are extra wrong-branch commits to handle first
git switch feat/x                     # or: git switch -c feat/x
git cherry-pick <sha>                 # use the SHA you noted, not the branch name
git switch main
git reset --hard HEAD~1               # ⚠ destructive; the previous --oneline check is the precondition
```

**Verify:**
```bash
git log --oneline -3 main             # bad commit gone
git log --oneline -3 feat/x           # bad commit landed here as new SHA
```

**Gotcha:** if you already pushed to `main`, **do not reset main**. Use revert (§6.4) and cherry-pick the original commit to `feat/x` instead.

**Recovery:** if reset removes more than you wanted, `git reflog` → find the pre-reset HEAD → `git reset --hard HEAD@{1}` brings it back. See §5.8.

---

### 6.2 I want to undo my last commit (keep my changes)

**What's happening:** Last commit was wrong, but the file edits are fine — you want to redo the commit.

**Run this:**
```bash
git reset --soft HEAD~1     # commit gone, changes still staged
# edit, restage as needed
git commit -m "better message"
```

**Verify:**
```bash
git status                  # should show staged changes
git log --oneline -3
```

---

### 6.3 I want to undo my last commit (discard my changes)

**What's happening:** Last commit was wrong AND the changes are bad — throw it all away.

**Run this:**
```bash
git status                  # confirm nothing else uncommitted you'd lose
git reset --hard HEAD~1     # ⚠ destructive
```

**Verify:**
```bash
git log --oneline -3        # the bad commit is gone
git status                  # clean
```

**Gotcha:** if you panic, `git reflog` → find the lost commit → `git reset --hard 'HEAD@{1}'` brings it back (quote braces in zsh).

---

### 6.4 I want to undo a commit that's already pushed

**What's happening:** Bad commit is on a shared branch (or main). Rewriting history is unsafe.

**Run this:**
```bash
git revert <sha>            # creates a new commit that inverts the bad one
git push
```

**For merge commits**, plain `git revert <merge-sha>` errors. You must pick a parent:
```bash
git revert -m 1 <merge-sha>   # -m 1 = keep first-parent line (the branch that received the merge)
```

**Verify:**
```bash
git log --oneline -3        # original commit + revert commit both visible
```

**Gotcha:** revert only inverts the diff. If the bad commit added a file, revert deletes it. If it changed config, revert restores old config. State that depends on the bad commit (db migrations, deployed artifacts) is **not** undone.

---

### 6.5 I want to fix the last commit message

**Not yet pushed:**
```bash
git commit --amend          # opens editor for new message
```

**Already pushed (your own feature branch):**
```bash
git commit --amend
git push --force-with-lease   # ⚠ never on main/shared
```

**Already pushed to main / shared branch:** leave it. Don't rewrite history others may have pulled.

---

### 6.6 I accidentally committed a secret or wrong file

**Not yet pushed (last commit only):**
```bash
git rm --cached <file>             # untrack but keep on disk
echo "<file>" >> .gitignore
git commit --amend --no-edit
```

**Already pushed:** the secret is **already exposed**. Order:
1. **Rotate the secret immediately** (regenerate token, change password, etc.). History rewrites alone do not protect a leaked credential.
2. Then scrub history with [git-filter-repo](https://github.com/newren/git-filter-repo):
   ```bash
   brew install git-filter-repo                   # or: pip install git-filter-repo
   git filter-repo --path <file> --invert-paths   # ⚠ rewrites entire history
   git push --force                               # ⚠ rewrites remote — see note below
   ```
   *Note:* this is the one place the bible uses bare `--force` instead of `--force-with-lease` (§1). `filter-repo` deliberately rewrites every ref; lease checks fight that intent and can leave dangling refs holding the secret. Use bare `--force` here knowingly.
3. Tell collaborators to re-clone. Their local clones still have the secret.

**Gotcha:** GitHub also caches commits in PRs and forks. Even after force-push, the SHA may still be reachable via direct URL. This is why rotation comes **first**.

---

### 6.7 My PR needs the latest changes from main

**Run this:**
```bash
git switch main
git pull
git switch feat/x
git rebase main             # replay your commits on top of new main
# resolve any conflicts (see §6.11)
git push --force-with-lease # ⚠ force needed because rebase rewrote your SHAs
```

**Alternative (if branch is shared):**
```bash
git switch feat/x
git merge main              # creates a merge commit; no force-push needed
git push
```

**Gotcha:** if the PR has been reviewed and you want to preserve review threads, prefer `merge` over `rebase` — rebases can detach review comments from the commits they were attached to. The GitHub PR UI's **"Update branch"** button defaults to a merge and is the safest one-click option for a shared PR.

---

### 6.8 I need to combine my last 3 commits into 1

**Run this:**
```bash
git rebase -i HEAD~3
# In the editor: leave first commit as `pick`, change others to `squash` (or `s`)
# Save, then edit the combined message in the next editor view
```

**If already pushed (your own branch only):**
```bash
git push --force-with-lease   # ⚠
```

**Gotcha:** `fixup` (or `f`) is like `squash` but discards the squashed commits' messages. Useful when the messages were just "wip" or "fix".

---

### 6.9 I need to split one big commit into smaller commits

**Run this:**
```bash
git rebase -i HEAD~N        # where N puts the big commit in range
# Mark the commit `edit` (or `e`) and save
# Rebase pauses at that commit:
git reset HEAD~              # un-commit but keep changes in working dir
git add -p                   # stage one logical chunk
git commit -m "first piece"
git add -p
git commit -m "second piece"
# repeat until clean
git rebase --continue
```

**Verify:**
```bash
git log --oneline -10
```

---

### 6.10 I'm mid-rebase and want out

**Run this:**
```bash
git rebase --abort
```

**Verify:**
```bash
git status                  # back to pre-rebase state
git log --oneline -5
```

**Gotcha:** works only if rebase is still in progress (`.git/rebase-merge/` or `.git/rebase-apply/` exists). After `--continue` finishes the rebase, `--abort` is no longer available — recover via reflog (§5.8): find the pre-rebase HEAD with `git reflog`, then `git reset --hard 'HEAD@{N}'`.

---

### 6.11 I have a merge conflict and don't know what to do

**What's happening:** Git can't auto-merge two changes to the same lines.

**Run this:**
```bash
git status                  # lists files marked "both modified"
# Open each conflicted file. Find conflict markers:
#   <<<<<<< HEAD
#   your version
#   =======
#   their version
#   >>>>>>> branch-name
# Edit to the final version you want. Remove all markers.
git add <resolved-file>
# When all conflicts resolved:
git merge --continue        # if mid-merge
git rebase --continue       # if mid-rebase
git cherry-pick --continue  # if mid-cherry-pick
```

**Tools that help:**
```bash
git mergetool               # opens a 3-way merge tool (configurable)
git diff                    # shows conflict hunks before resolving
git checkout --ours <file>  # take your version wholesale
git checkout --theirs <file> # take their version wholesale
```

**Gotcha:** during a **rebase**, "ours" and "theirs" are swapped relative to merge — `--ours` means the branch you're rebasing onto, `--theirs` means your commits. This trips everyone up.

---

### 6.12 I deleted a branch I needed

**Run this:**
```bash
git reflog                  # find the last commit on that branch
# Look for "checkout: moving from <branch>" or commits with that branch's tip
git switch -c <branch> <sha>
```

**Verify:**
```bash
git log --oneline -5
```

**Gotcha:** if the branch was never checked out locally and only existed on the remote, `git fetch` and check `git branch -a` — it may still be on the remote.

---

### 6.13 I want a commit from another branch without merging the whole branch

**Run this:**
```bash
git log --oneline other-branch    # find the SHA
git switch feat/x
git cherry-pick <sha>
```

**Multiple commits:**
```bash
git cherry-pick <sha1> <sha2> <sha3>
git cherry-pick <sha1>..<sha2>     # range, exclusive of sha1
```

**Gotcha:** cherry-pick creates a **new commit with a different SHA**. If the original branch later merges into yours, you'll have the change twice (usually with no conflict, since the diffs are identical, but the history is messy).

**Range inclusivity:** `<sha1>..<sha2>` is exclusive of `<sha1>`. To include it, use `<sha1>^..<sha2>`.

---

### 6.14 I pushed to main by mistake

**You haven't yet been overwritten by anyone:**
```bash
git revert <sha>            # creates an undo commit
git push
```

**You broke production:** revert immediately as above. Then root-cause separately. Do **not** force-push main.

---

### 6.15 Two machines have diverged on the same branch

**What's happening:** You committed on Mac, then committed on Windows without pulling first. Both have local commits the other doesn't.

**Run this** (on the machine you're currently on):
```bash
git fetch
git status                  # will say "diverged"
git pull --rebase           # replays your local commits on top of remote's
# resolve conflicts if any (§6.11)
git push
```

**Or, to merge instead:**
```bash
git pull                    # creates a merge commit
git push
```

**Gotcha — force-pushed from the other machine:** if you ran `git push --force-with-lease` on machine A (e.g. after a rebase), the remote SHA changed. On machine B, `git pull --rebase` will replay your local commits onto the *new* tip and may surface phantom conflicts. If you're certain the other machine's branch is canonical:
```bash
git fetch
git reset --hard origin/<branch>   # ⚠ discards local commits; only safe if you have no local work
```

**Prevention:** end every machine's session with `git push`, and start every machine's session with `git pull`. See [[Multi-Instance Protocol]].

---

### 6.16 My worktree is in a weird state

**Symptoms:** worktree path is gone but `git worktree list` still lists it; or you can't switch branches in the worktree because Git says it's checked out elsewhere.

**Run this:**
```bash
git worktree list           # see what Git thinks exists
git worktree prune          # remove records for worktrees whose dirs are gone
git worktree remove <path>  # remove a specific one
git worktree remove --force <path>   # ⚠ if dirty
```

**Claude Code worktrees** live at `~/Documents/Development/<repo>/.claude/worktrees/<adjective-name>` with branches named `claude/<adjective-name>`. They **persist after the task ends** — they accumulate over time. Prune periodically:
```bash
git worktree list                                    # audit what's still around
git worktree remove .claude/worktrees/<adjective-name>  # remove one
git worktree prune                                   # clean records for already-deleted dirs
```
See [[Claude Code Setup]].

---

### 6.17 I need to remove a file from all of git history

**Use case:** giant file accidentally committed long ago, blowing up clone size. Or a leaked secret (but **rotate first** — see §6.6).

**Run this** (with [git-filter-repo](https://github.com/newren/git-filter-repo)):
```bash
brew install git-filter-repo                         # or: pip install git-filter-repo
git filter-repo --path path/to/file --invert-paths   # ⚠ rewrites all history
git push --force --all                               # ⚠ bare --force (not --force-with-lease) is intentional here — see §6.6
git push --force --tags                              # ⚠ rewrites tags
```

**Preconditions:**
- All collaborators warned and ready to re-clone.
- Backup of the repo exists (`cp -r repo repo.bak` first).
- You are not on `main`/`master` of a high-traffic repo without team coordination.

**Gotcha:** `git filter-branch` is the older built-in but is deprecated and slow. Always use `git-filter-repo`.

---

### 6.18 `gh pr create` failed because branch isn't pushed

**Run this:**
```bash
git push -u origin <current-branch>
gh pr create --title "..." --body "..."
```

`-u` only needed the first time per branch. After that, plain `git push` works.

---

### 6.19 PR has conflicts and I'm scared to resolve them

**Option A — locally (recommended):**
```bash
gh pr checkout <pr-number>      # switches you to the PR's branch
git fetch origin main
git rebase origin/main          # or: git merge origin/main
# resolve conflicts (§6.11)
git push --force-with-lease     # ⚠ rebase only; plain push if you merged
```

**Option B — in GitHub UI:** click "Resolve conflicts" on the PR. Only works for simple conflicts. If you don't trust the diff visible there, use Option A.

**Gotcha:** never resolve conflicts by accepting "all theirs" or "all ours" without reading the actual diff. Conflicts mean both sides changed the same lines for a reason.

---

### 6.20 I want to see what changed between two refs

| Goal | Command |
|---|---|
| Diff between two commits | `git diff <sha1>..<sha2>` |
| Diff between two branches | `git diff main..feat/x` |
| Just the file list | `git diff --name-only main..feat/x` |
| Just the stat summary | `git diff --stat main..feat/x` |
| Commits in branch B but not A | `git log A..B --oneline` |
| Commits unique to each | `git log A...B --oneline --left-right` (note **3 dots**) |
| Compare two PRs | `gh pr diff <pr-number>` |

---

### 6.21 I need to find when a bug was introduced (`git bisect`)

**Manual:**
```bash
git bisect start
git bisect bad                  # current commit is broken
git bisect good <known-good-sha>
# Git checks out a midpoint commit. Test it.
git bisect good                 # or: git bisect bad
# Repeat until Git names the first bad commit.
git bisect reset                # return to your original HEAD
```

**Automated:**
```bash
git bisect start HEAD <good-sha>
git bisect run ./test.sh        # exit 0 = good, non-zero = bad
# Git finds the first bad commit automatically
git bisect reset
```

**Gotcha:** `bisect run` requires a deterministic, fast test. If the test is flaky, bisect produces wrong results.

---

## 7. Worktree Workflows (Joe-specific)

Claude Code creates a worktree per task at:
```
~/Documents/Development/<repo>/.claude/worktrees/<adjective-name>
```

Each worktree has its own branch (e.g. `claude/<adjective-name>`) but shares the underlying `.git` with the main checkout.

### When to use worktrees instead of branches
- You want to keep `main` checked out in one window for builds/testing while editing in another
- You need to run a long build in one branch and continue editing in another
- Claude Code is doing parallel work — each task gets isolated files

### Manual worktree (without Claude Code)
```bash
cd ~/Documents/Development/<repo>
git worktree add ../<repo>-feat-x feat/x          # existing branch
git worktree add -b feat/y ../<repo>-feat-y main  # new branch off main
```

### Cleanup
```bash
git worktree list
git worktree remove ~/Documents/Development/<repo>-feat-x
git worktree prune                                # for stale entries
```

### Rules
- **Never delete a worktree directory with `rm -rf`** without first running `git worktree remove`. Leaves Git's records inconsistent. If you already did, run `git worktree prune` to clean up.
- Each worktree can only have a given branch checked out **once across all worktrees**. If you try to switch a worktree to a branch that's checked out elsewhere, Git refuses.

---

## 8. Multi-Machine Workflows (Mac ↔ Win)

See [[Multi-Instance Protocol]] for the vault-side handoff schema. For code:

### Golden flow
| When | Run |
|---|---|
| Starting work on machine A | `git fetch && git pull` on every active branch you might touch |
| Pausing work | `git status` (verify clean) → `git push` |
| Resuming on machine B | `git fetch && git pull` |

### Handling divergence
If you forget to push on machine A and start working on machine B:

```bash
# On machine B, after fetch:
git status                  # "diverged"
git pull --rebase           # cleanest history, your B commits on top
# OR
git pull                    # merge commit; safer if rebase scares you
```

If both machines have committed unique changes that conflict, see §6.15 + §6.11.

### Stash hand-off (rare, when you don't want to commit yet)
Stash doesn't sync via Git. Either:
- Commit a "wip:" commit and revert it on the other machine, or
- Copy the diff manually: `git diff > /tmp/wip.patch`, transfer the file, `git apply /tmp/wip.patch` on the other machine.

---

## 9. PR Workflow with `gh` CLI

### Create
```bash
gh pr create --title "feat: short summary" --body "$(cat <<'EOF'
## Summary
- bullet 1
- bullet 2

## Test plan
- [ ] manual test step
- [ ] CI passes
EOF
)"
```

### Inspect
| Command | Purpose |
|---|---|
| `gh pr list` | All open PRs in this repo |
| `gh pr list --author @me` | Just yours |
| `gh pr view` | Current branch's PR |
| `gh pr view <num>` | Specific PR |
| `gh pr view <num> --web` | Open in browser |
| `gh pr diff <num>` | Show the diff |
| `gh pr checks` | CI status for current branch's PR |

### Switch to a PR locally (e.g. to review)
```bash
gh pr checkout <num>        # creates/switches to a local branch tracking the PR
```

### Comment / review
```bash
gh pr comment <num> --body "LGTM"
gh pr review <num> --approve
gh pr review <num> --request-changes --body "see comments"
```

### Merge
```bash
gh pr merge <num> --squash --delete-branch    # solo workflow default
gh pr merge <num> --merge                     # preserves all commits
gh pr merge <num> --rebase                    # rebases onto base, no merge commit
```

### Drafts
```bash
gh pr create --draft --title "..." --body "..."
gh pr ready <num>           # convert draft → ready for review
```

### Useful one-off API calls
```bash
gh api repos/{owner}/{repo}/pulls/<num>/comments    # all review comments on a PR
gh api repos/{owner}/{repo}/issues/<num>/comments   # general PR comments
```
`{owner}` / `{repo}` / `{branch}` are the documented placeholders `gh api` expands from current repo context. The older `:owner/:repo` colon form sometimes still works but is undocumented — don't rely on it.

---

## 10. Co-Authored Commits with Claude

When Claude Code drives a commit, the convention (from `~/.claude/CLAUDE.md`) is to add a `Co-Authored-By` trailer. Use a heredoc so the multiline message renders correctly:

```bash
git commit -m "$(cat <<'EOF'
feat(audio): add onset detector hints

Why: tempo and valence cues need a single source of truth across
web and iOS. This wires the C2 detector to both targets.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

**When to add the trailer:** when Claude wrote substantive code in the commit. Pure mechanical edits (rename, formatter run) don't need it.

**When to skip:** purely human commits, version bumps, merge commits.

---

## 11. Glossary & Authoritative Sources

### Terms
| Term | Meaning |
|---|---|
| **HEAD** | Pointer to the current commit (usually via a branch) |
| **detached HEAD** | HEAD points directly at a commit, not a branch — commits made here are easy to lose |
| **fast-forward (FF)** | Merge that just moves the branch pointer because the target is an ancestor of the source |
| **upstream** | The remote branch your local branch tracks |
| **origin** | Convention for the default remote |
| **fork** | A copy of a repo on GitHub under your account; you push to your fork, then PR upstream |
| **squash** | Combine multiple commits into one |
| **rebase** | Replay commits on top of a different base |
| **cherry-pick** | Apply one commit's diff onto current branch as a new commit |
| **revert** | Create a new commit that inverts another commit's diff |
| **reflog** | Log of every HEAD movement; the undo history |
| **dangling commit** | A commit no branch/tag references; reachable only via reflog or `fsck` until garbage-collected |
| **annotated tag** | Tag stored as a full Git object with author/date/message; preferred for releases |
| **lightweight tag** | Tag that's just a name pointing at a commit; for personal markers |

### Authoritative sources
- **Pro Git book** (free, official): https://git-scm.com/book
- **`git help <command>`** or **`git <command> --help`** — the man pages are authoritative
- **GitHub CLI manual**: https://cli.github.com/manual/
- **git-filter-repo docs**: https://github.com/newren/git-filter-repo
- **GitHub Docs**: https://docs.github.com

---

## Notes
