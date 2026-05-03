# 🌳 Git — Scenario-Based Interview Questions

---

## 🟢 Branching, Merging & Conflicts

---

**Q1. [L1] You started coding directly on `main` and made three commits. You realize this work belongs on a feature branch and you haven't pushed yet. How do you move those commits to a new branch and clean up `main`?**

> *What the interviewer is testing:* Basic branch mechanics, understanding that branches are just movable pointers to commits.

**Answer:**
A branch in Git is just a pointer to a commit, so "moving" commits is really just moving pointers around.

1. **Create the feature branch from where you are right now** — this captures your three commits on the new branch:
   ```bash
   git branch feature/my-work
   ```
2. **Reset `main` back to where the remote is** — your local `main` still points at the third commit; rewind it:
   ```bash
   git reset --hard origin/main
   ```
3. **Switch to the feature branch and continue:**
   ```bash
   git checkout feature/my-work
   ```

The order matters: create the new branch *before* you reset, otherwise the commits become unreachable from any branch (they'd still be in `reflog` for ~90 days, but don't rely on that). Since you never pushed, no one else is affected.

---

**Q2. [L1] What is the difference between `git clone` and `git fork`, and when would you use each in practice?**

> *What the interviewer is testing:* Understanding of remote workflows and contribution models.

**Answer:**
`git clone` is a Git command — `git fork` is **not** a Git command, it's a GitHub/GitLab feature.

- **Clone** (`git clone <url>`) — downloads a copy of a repo to your laptop. You get the full history and a remote called `origin` pointing back to wherever you cloned from. Use it when you have push access to the repo.
- **Fork** — creates a server-side copy of the repo under your own account on GitHub/GitLab. Then you clone *your fork* locally. Use it when you don't have push access to the original (typical for open-source contributions).

The contribution flow for an open-source project is: fork on GitHub → clone your fork → add the original repo as `upstream` → push branches to your fork → open a PR from your fork to the original. For an internal company repo where everyone has write access, just clone directly and push branches.

---

**Q3. [L2] You ran `git merge` and Git stopped with conflicts in three files. Walk me through exactly how you resolve them, and explain what the conflict markers mean.**

> *What the interviewer is testing:* Practical conflict resolution, understanding of three-way merge.

**Answer:**
Git stopped because it couldn't auto-merge — both branches changed the same lines.

1. **See what's conflicted:**
   ```bash
   git status   # files marked "both modified"
   ```
2. **Open each file.** You'll see markers like:
   ```
   <<<<<<< HEAD
   your version (current branch)
   =======
   their version (incoming branch)
   >>>>>>> feature/x
   ```
   `HEAD` is what's on the branch you're merging *into*. The lines below `=======` are from the branch you're merging *in*. The `|||||||` (if `merge.conflictstyle = diff3` is set) shows the common ancestor — extremely useful for understanding intent.
3. **Edit the file** to the correct final state and remove all `<<<`, `===`, `>>>` markers. Don't just pick a side blindly — re-read both intents.
4. **Mark resolved and finish:**
   ```bash
   git add <file>
   git merge --continue   # or: git commit
   ```
5. If you panic, `git merge --abort` puts you back where you started.

For repeated conflicts on the same hunk in long-lived branches, enable `git rerere` so Git remembers your resolution next time.

---

**Q4. [L2] When should you use `git rebase` instead of `git merge`, and what is the one rule you must never break?**

> *What the interviewer is testing:* Workflow tradeoffs, understanding the danger of rewriting shared history.

**Answer:**
Both integrate changes from one branch into another, but they produce very different histories.

- **Merge** preserves the actual branching history (creates a merge commit). Good for `main`/release branches where the history of *when things converged* matters.
- **Rebase** replays your commits on top of the target branch, producing a linear history with no merge commits. Good for cleaning up your local feature branch *before* opening a PR — it makes review easier and `git log` readable.

**The one rule: never rebase commits that have been pushed and that other people are working on.** Rebase rewrites commit hashes. If a teammate has pulled the old commits and you force-push rewritten ones, their next pull is a mess and they may re-introduce the old commits. Rebase your private branch all you want; never rebase shared branches like `main` or `develop`.

Common workflow: rebase your feature branch onto latest `main` (`git pull --rebase origin main`), squash interactively (`git rebase -i`), force-push to your *own* feature branch, then merge the PR.

---

## 🔵 Undo, Recovery & History Rewriting

---

**Q5. [L1] You have local changes in your working directory that you don't want anymore. How do you discard them, and what's the difference between discarding tracked vs untracked changes?**

> *What the interviewer is testing:* Understanding of the working tree, staging area, and untracked files.

**Answer:**
Different commands for different states — and one of them is destructive, so know which:

- **Modified tracked files (not staged):** `git restore <file>` (or older syntax `git checkout -- <file>`) reverts the file to what's in `HEAD`.
- **Staged changes:** `git restore --staged <file>` unstages, leaving the modification in the working tree. Add another `git restore <file>` to discard it too.
- **Untracked (new) files:** `git restore` won't touch them — Git doesn't know they exist. Use `git clean -fd` to delete them. Run `git clean -nd` first to preview what will be deleted.
- **Nuke everything back to a clean checkout:** `git reset --hard HEAD && git clean -fd`.

These are destructive — there's no undo for working-tree changes that were never committed. When unsure, `git stash` first; you can drop the stash later if you don't need it.

---

**Q6. [L1] You added `config.yaml` to `.gitignore` but Git is still tracking it and showing it as modified. Why, and how do you fix it?**

> *What the interviewer is testing:* Understanding that `.gitignore` only affects untracked files.

**Answer:**
`.gitignore` only prevents *new* files from being tracked. It does **not** untrack files that are already in the repo. Git is still watching `config.yaml` because it was committed before the ignore rule existed.

Fix it by removing the file from the index while keeping it on disk:

```bash
git rm --cached config.yaml
git commit -m "Stop tracking config.yaml"
```

The `--cached` flag is critical — without it, `git rm` deletes the local file too. After the commit, `.gitignore` will keep it out going forward.

For a whole directory: `git rm -r --cached path/to/dir`. To re-check what Git is currently tracking that *should* be ignored: `git ls-files -ci --exclude-standard`.

---

**Q7. [L1] You're in "detached HEAD" state. What does that mean, and how do you get out of it without losing work?**

> *What the interviewer is testing:* Mental model of HEAD, branches, and commits.

**Answer:**
Normally `HEAD` points to a branch (e.g. `main`), and the branch points to a commit. "Detached HEAD" means `HEAD` points directly at a commit with no branch in between — usually because you ran `git checkout <commit-hash>` or `git checkout v1.2.0` (a tag).

It's not broken, just risky: any new commits you make in this state aren't on a branch. If you switch away, those commits become unreachable and will eventually be garbage-collected.

To recover:

- **No new commits made yet?** Just `git switch main` (or whichever branch). No data at risk.
- **Made commits you want to keep?** Create a branch from where you are *before* switching:
  ```bash
  git switch -c rescue-branch
  ```
  Now those commits are anchored to a branch and safe.
- **Already switched away and panicking?** `git reflog` shows everywhere `HEAD` has been; find your commit hash and `git branch rescue-branch <hash>`.

---

**Q8. [L2] You ran `git push --force` to your feature branch but realized you wiped out a teammate's commit they had pushed five minutes earlier. How do you recover their commit, and how do you prevent this next time?**

> *What the interviewer is testing:* Reflog recovery, `--force-with-lease`, awareness of force-push hazards.

**Answer:**
The commit isn't lost on the remote yet — it's still in the remote's reflog and likely in your teammate's local repo. Recovery:

1. **Ask the teammate** — their local branch still has the commit. They can push it back (after pulling your changes and rebasing it on top), or share the hash.
2. **Check the remote's reflog** if you have shell access to the server (rare on managed services). On GitHub, the events API and "Network" graph sometimes still show the dangling commit hash for a while; you can fetch it directly: `git fetch origin <sha>`.
3. **Check your own clone** — if you fetched their commit before force-pushing, `git reflog` on the remote-tracking branch (`git reflog show origin/feature-x`) will list it. Cherry-pick it back: `git cherry-pick <sha>` then push.

**Prevention:**

- Use `git push --force-with-lease` instead of `--force`. It refuses to push if the remote has commits you haven't seen — exactly this scenario.
- Better: `git push --force-if-includes` (Git 2.30+) — also verifies your local ref includes the latest fetch.
- Branch protection rules on `main`/`develop` should block force-push entirely. Force-push should only be allowed (and only via `--force-with-lease`) on personal feature branches.

---

**Q9. [L2] You merged a PR into `main` that turned out to be broken in production. The merge has been there for two hours and 15 commits have landed since. How do you back it out safely?**

> *What the interviewer is testing:* `git revert` on merge commits, understanding that you can't just delete published history.

**Answer:**
You can't `reset` `main` — it's shared and 15 other commits have built on top. The right tool is `git revert`, which creates a *new* commit that undoes the changes.

For a merge commit you need `-m` to tell Git which parent to revert to (the "mainline"):

```bash
git revert -m 1 <merge-commit-sha>
```

`-m 1` means "treat parent 1 (the `main` side) as the mainline, and revert everything that came in from the feature side." `-m 2` would do the opposite.

Push the revert. Now `main` is back to a working state and the 15 unrelated commits are preserved.

**Gotcha:** if later you want to re-merge the fixed version of that feature branch, a plain merge will appear to do nothing because Git thinks those changes are already on `main` (they were, then reverted). You either revert the revert (`git revert <revert-sha>`) before re-merging, or rebase the feature branch onto current `main` so the commits get fresh hashes.

---

**Q10. [L2] You have eight messy "WIP" / "fix typo" / "more fixes" commits on your feature branch. Reviewers want to see one clean commit per logical change. How do you reshape the history before opening the PR?**

> *What the interviewer is testing:* Interactive rebase, squash/fixup workflow.

**Answer:**
Use interactive rebase. Pick a base (usually where your branch diverged from `main`):

```bash
git rebase -i origin/main
```

An editor opens with one line per commit:

```
pick a1b2c3 Add user model
pick d4e5f6 WIP
pick 7g8h9i fix typo
pick j0k1l2 Add user controller
pick m3n4o5 more fixes
...
```

Change `pick` to:

- `squash` (or `s`) — combine into the previous commit and let you edit the message.
- `fixup` (or `f`) — same but discard this commit's message (great for "fix typo" commits).
- `reword` (or `r`) — keep the commit but rewrite its message.
- `drop` (or `d`) — remove the commit entirely.
- Reorder lines to reorder commits.

Save and exit. Git replays the commits according to your plan; resolve any conflicts and `git rebase --continue`.

Force-push your *own* feature branch with `--force-with-lease`. Never do this on `main` or any branch others have based work on.

Pro tip: as you go, commit with `git commit --fixup=<sha>` and finish with `git rebase -i --autosquash` — the editor pre-arranges fixups for you.

---

**Q11. [L3] A junior engineer ran `git reset --hard HEAD~5` on their local branch and lost five commits of in-progress work. Nothing was pushed. Walk me through the recovery.**

> *What the interviewer is testing:* Deep understanding of refs, reflog, and Git's garbage collection model.

**Answer:**
A `--hard` reset moves the branch pointer and discards working-tree changes — but the commits themselves are not deleted. They're orphaned (no branch points to them) and will sit in the object database until garbage collection runs (default: ~30 days for unreachable objects, 90 days if reachable from reflog).

Recovery via the reflog, which is a per-clone log of every move `HEAD` (and each branch ref) has made:

```bash
git reflog                       # shows HEAD movements
# or, more targeted:
git reflog show feature-branch   # shows that branch's history
```

You'll see something like:

```
abc1234 HEAD@{0}: reset: moving to HEAD~5
def5678 HEAD@{1}: commit: Last good work
...
```

`def5678` is the tip of the work that was wiped. Restore it:

```bash
git reset --hard def5678
# or, safer if you're unsure:
git branch rescue def5678
git switch rescue
```

Two important caveats: (1) **the reflog is local** — it doesn't help if the commits never existed on this clone (e.g. teammate's machine). (2) Run recovery *before* `git gc` runs. If you suspect a teammate already ran `git gc --prune=now`, the objects may genuinely be gone. As a habit, alias `reset --hard` to require confirmation, and teach the team `git stash` and `git switch -c backup` before risky operations.

---

## 🟡 Collaboration & Remote Workflows

---

**Q12. [L1] In one paragraph: what does `git pull` actually do, and why do some teams prefer `git pull --rebase`?**

> *What the interviewer is testing:* Understanding of fetch + merge composition and history shape preferences.

**Answer:**
`git pull` is two commands stitched together: `git fetch` (download new commits and tags from the remote into `origin/<branch>`) followed by `git merge origin/<branch>` into your current branch. If your local branch has commits the remote doesn't, that merge produces a merge commit ("Merge branch 'main' of …"), which clutters history with bookkeeping commits that don't represent real work.

`git pull --rebase` replaces the merge step with a rebase: your local commits are temporarily set aside, the new remote commits are applied first, and then your commits are replayed on top. Result: a clean linear history with no noise commits. Many teams set `git config --global pull.rebase true` so this becomes the default. The tradeoff is that rebase rewrites your *local* commit hashes — fine for unpushed work, but you should never rebase commits others have already pulled.

---

**Q13. [L1] Explain the four states a file can be in inside a Git repo: untracked, modified, staged, committed. Why does Git have a separate staging area?**

> *What the interviewer is testing:* Mental model of the index/staging area.

**Answer:**
A file moves through these states:

- **Untracked** — exists in your working directory but Git has never been told about it. Shows up under "Untracked files" in `git status`.
- **Modified (tracked)** — Git knows about the file, and the working-tree version differs from what's in the last commit. Shows under "Changes not staged for commit."
- **Staged** — you ran `git add <file>`, copying its current content into the *index* (staging area). Shows under "Changes to be committed." `git commit` will record exactly this snapshot.
- **Committed** — the staged content has been written into a commit object. Working tree, index, and `HEAD` all agree.

The staging area exists so you can build the *next* commit deliberately instead of having every saved file immediately become part of it. You can edit ten files, but `git add` only the three related ones, then `git commit` a focused logical change. `git add -p` takes this further — letting you stage individual hunks within a file. Without the index, you'd have to commit everything at once or stash/branch around it.

---

**Q14. [L2] A critical bug-fix commit exists on `develop` and you need exactly that one commit on a release branch — without bringing along the other 50 commits on `develop`. How do you do it, and what could go wrong?**

> *What the interviewer is testing:* Cherry-pick mechanics and its hazards.

**Answer:**
This is `git cherry-pick`:

```bash
git switch release/1.4
git cherry-pick <commit-sha>
git push
```

Cherry-pick takes the diff that commit introduced and applies it as a *new* commit on the current branch (new SHA, same content). For multiple commits: `git cherry-pick <sha1> <sha2> <sha3>` or a range `git cherry-pick A..B`.

What can go wrong:

- **Conflicts** — if the surrounding code on `release/1.4` is different from `develop`, the patch may not apply cleanly. Resolve, then `git cherry-pick --continue`.
- **Missing dependencies** — the commit may rely on a refactor or a helper that was introduced in an earlier commit on `develop` but isn't on `release/1.4`. The cherry-pick may apply but the code won't compile or behave correctly. You may need to cherry-pick the prerequisite commit too, or write a backport patch.
- **Duplicate-looking commits** — when `develop` is later merged into `release` (or vice versa), Git usually figures out the commits are equivalent (via patch-id), but in some workflows you can end up with two commits doing the same thing under different SHAs. Use `git cherry-pick -x` to record "(cherry picked from commit …)" in the message — invaluable later for traceability.

For ongoing back-port flows (fix on `main`, port to `release/*`), some teams prefer a dedicated `hotfix/*` branch that's merged into both, which avoids cherry-picks entirely.

---

**Q15. [L2] Production is broken. You know it worked at the v2.3 tag but not at HEAD, with about 200 commits between them. How do you find the exact commit that introduced the bug efficiently?**

> *What the interviewer is testing:* `git bisect` workflow, ability to automate root-cause analysis.

**Answer:**
Use `git bisect` — a binary search over commits.

```bash
git bisect start
git bisect bad HEAD       # tell Git the current commit is broken
git bisect good v2.3      # tell Git this old tag was fine
```

Git checks out a commit roughly halfway between them. You build/test it and report:

```bash
git bisect good   # if the bug isn't there
git bisect bad    # if the bug is there
```

Each step halves the search space. With 200 commits, you'll converge in ~8 steps (`log2(200)`). When done, Git prints "X is the first bad commit." Then:

```bash
git bisect reset   # restore your branch
```

**Automate it** if you have a reproducer script that exits 0 (good) or non-zero (bad):

```bash
git bisect start HEAD v2.3
git bisect run ./scripts/reproduce-bug.sh
```

Walk away — Git does the rest.

Tips: skip commits that don't build with `git bisect skip`; use `--first-parent` if your history has many merge commits and you only care about which *merge* introduced the bug. Bisect works because every commit is an immutable snapshot, so each one is independently testable.

---

**Q16. [L2] A developer accidentally committed an AWS access key and pushed it to the public repo. The team noticed 30 minutes later. What's the correct response, in priority order?**

> *What the interviewer is testing:* Incident response thinking; understanding that Git history rewriting alone is not enough.

**Answer:**
Treat the credential as compromised the moment it touches a public surface. Order matters:

1. **Rotate the credential first, before anything else.** In AWS IAM, deactivate the leaked key and issue a new one. Anything you do to Git history is secondary — the key was public for 30 minutes, scrapers are constant, and assume it was harvested.
2. **Audit usage.** Check CloudTrail for any calls authenticated with that key — region, source IP, services touched. If anything looks suspicious, escalate to security.
3. **Remove the secret from history.** A plain `git revert` is **not enough** — the file is still in old commits in `.git/objects` and visible on GitHub forever. Use `git filter-repo` (the modern replacement for `filter-branch`):
   ```bash
   git filter-repo --path secrets.env --invert-paths
   ```
   or `--replace-text` to redact a string everywhere. Then force-push (this rewrites history; coordinate with the team).
4. **Invalidate forks and caches.** GitHub caches the SHA — open a support ticket asking them to purge the leaked commit, and tell anyone with a fork to re-clone.
5. **Add prevention.** Pre-commit hook with `gitleaks` or `detect-secrets`, plus push protection / secret scanning enabled at the org level so this is blocked next time.

The order is non-negotiable: rotate → audit → scrub → prevent. Reversing 1 and 3 is a common mistake — you can't un-leak a key, but you can stop it from being valid.

---

## 🔴 Advanced

---

**Q17. [L3] Your engineering org has grown to 50 developers across three time zones, all working on a single backend service. You currently use long-lived `develop`/`feature/*` branches with weekly merges to `main`. Releases are painful and conflict-heavy. How would you change the branching strategy, and what tradeoffs are you accepting?**

> *What the interviewer is testing:* Strategic thinking on branching models for scale.

**Answer:**
Long-lived feature branches don't scale — the longer a branch lives, the more it diverges, and merges become expensive. The three viable options:

- **Git Flow** (`main`, `develop`, `release/*`, `hotfix/*`, `feature/*`) — heavy ceremony, originally designed for shrink-wrapped software with explicit version numbers. For a fast-moving SaaS backend, this is overkill and a step in the wrong direction.
- **GitHub Flow** — single `main`, short-lived feature branches, PR review, deploy from `main`. Light, simple, well understood.
- **Trunk-based development** — everyone commits to (or merges tiny PRs into) `main` at least daily. No long-lived branches. Incomplete features hide behind feature flags. Continuous deployment from `main`.

For a 50-engineer SaaS team with conflict pain, I'd push toward **trunk-based**:

- Mandate short-lived branches (≤ 24-48 hours from branch to merge).
- Require PR review + green CI to merge — branch protection enforced.
- Adopt feature flags so half-built features can land safely behind a toggle.
- Continuously deploy `main` to staging; promote to prod on a cadence (or per-merge once confident).

**Tradeoffs you're accepting:**

- Investment in a feature-flag platform (LaunchDarkly, Unleash, or homegrown) and the discipline to clean up stale flags.
- Stronger CI investment — `main` must always be releasable, which means fast and reliable test suites and probably required status checks, code coverage gates, etc.
- Cultural shift — engineers must break work into smaller increments, which some find harder than long branches.

What you gain: dramatically smaller merge conflicts (because everyone is integrating against the same recent `main`), faster lead time, simpler mental model, and the ability to ship hotfixes without coordinating across release branches.

---

**Q18. [L3] Your monorepo has grown to 25 GB and 10 years of history. New hires take 45 minutes to clone, IDE indexing is slow, and most engineers only need ~5% of the tree. What Git-side techniques would you use, and where do they fall short?**

> *What the interviewer is testing:* Understanding of partial clone, sparse-checkout, LFS, and their limits.

**Answer:**
Three orthogonal techniques solve three different problems:

1. **Partial clone** — defers downloading file blobs until they're actually accessed:
   ```bash
   git clone --filter=blob:none <url>
   ```
   Fetches all commits and trees but no file contents. When you `checkout` or `log -p` a path, Git lazily fetches just those blobs. Cuts initial clone from 25 GB to a few hundred MB. Requires a server that supports it (GitHub, GitLab, Azure DevOps, Gitea all do).

2. **Sparse-checkout** — limits which paths exist in your working tree:
   ```bash
   git sparse-checkout init --cone
   git sparse-checkout set services/payments libs/common
   ```
   You still have the full history, but only the directories you listed materialize on disk. IDE indexing now sees 5% of the tree. Combine with partial clone for maximum effect (`git clone --filter=blob:none --sparse <url>`).

3. **Git LFS** for large binaries — design files, ML model artifacts, video. LFS stores a small pointer file in Git and the actual blob on a separate object store, fetched on demand. Adopt this *going forward*; converting historical large blobs requires `git lfs migrate import` plus a force-push, which rewrites history.

**Where they fall short:**

- Partial clone makes operations like `git log -p`, `git blame`, and bisect across many files trigger on-demand fetches that can be slow or fail offline. Engineers on flaky networks suffer.
- Sparse-checkout doesn't help operations that *traverse* the repo — `git grep` over a sparse checkout misses files you didn't materialize, which can be surprising. Cone mode mitigates but doesn't eliminate this.
- LFS adds operational dependency on the LFS server, increases hosting cost, and storage isn't free or fast for huge blobs. It's not a silver bullet — for truly large binary pipelines (e.g. game asset workflows), Perforce or a content-addressed object store is sometimes a better fit than Git.
- None of these address the *root* problem on Git itself: extremely deep histories on a few hot files (think: lockfiles touched by everyone) can still slow `blame` and `log`. At true Google/Microsoft scale you eventually outgrow vanilla Git and look at VFS for Git, Scalar, or Piper-style virtual filesystems.

The pragmatic recipe for most teams: sparse + partial clone via a `git clone` wrapper script for new hires, LFS adopted for any binary > a few MB, and a "no committing build artifacts" lint in CI.

---

**Q19. [L3] Your company is enforcing supply-chain integrity. The CISO wants every commit on `main` to have a verifiable author and to be tamper-evident. How do you implement this, and what attack does it actually prevent?**

> *What the interviewer is testing:* Understanding of commit signing, identity vs authorship, and threat modeling.

**Answer:**
By default, Git's `Author` and `Committer` headers are unauthenticated free-text — anyone can run `git config user.email "ceo@company.com"` and produce commits that look like the CEO. Signing fixes that with cryptographic proof.

**Implementation:**

1. **Signing keys per developer.** Either GPG (traditional) or **SSH signing** (Git ≥ 2.34, much simpler — reuse the SSH key engineers already have):
   ```bash
   git config --global gpg.format ssh
   git config --global user.signingkey ~/.ssh/id_ed25519.pub
   git config --global commit.gpgsign true
   git config --global tag.gpgsign true
   ```
2. **Allowed-signers file** mapping email → public key, distributed via your IDP / SSO so trust is centralized, not per-laptop.
3. **Upload the public key to GitHub/GitLab** under "Signing key." Now the platform shows a "Verified" badge and exposes verification status via the API.
4. **Branch protection on `main`** — require signed commits, require linear history, require status checks. Block merges where any commit is unsigned.
5. **CI verification** — a pre-merge job that runs `git verify-commit` against each commit in the PR, failing if any commit isn't signed by a key in the allowed-signers list. Don't rely solely on the platform UI badge.
6. **CODEOWNERS** to require domain-expert review on sensitive paths (e.g. `/infra/`, `/auth/`, CI workflows themselves).

**What this actually prevents:**

- **Author spoofing** — an attacker who compromises one developer's laptop can no longer forge commits as a different developer (different key).
- **Tampering with merged history** — if someone with repo-admin access tries to silently rewrite a past commit, the signature on the original commit chain breaks and CI rejects it.
- **Compromised CI tokens** pushing arbitrary code as humans — unsigned commits are blocked, so a leaked PAT can't ship merge-able changes without a key.

**What it does *not* prevent:**

- A compromised developer laptop with their key on it — the attacker's commits will sign correctly. Mitigations: hardware-backed keys (YubiKey for GPG, or SSH keys in `ssh-agent` with confirmation), short-lived keys via SSO-issued certificates, and behavioral monitoring.
- Malicious code that's *legitimately* authored and signed (insider threat, social-engineered review). That's what CODEOWNERS, mandatory review, and SCA tooling are for.

Signing answers "who wrote this commit?" with cryptography. It does not answer "is this code safe?" — that's a separate problem.

---

**Q20. [L3] Design a branch-protection and merge-policy setup for a regulated environment (PCI / SOC 2) with 30 services in a monorepo. Walk me through the controls you'd put on `main` and how they interact with developer ergonomics.**

> *What the interviewer is testing:* Holistic policy design balancing compliance, security, and velocity.

**Answer:**
Compliance auditors essentially want to see: every change to `main` is reviewed by someone other than the author, every change is traced to a ticket, every change runs the same controls, and the history is tamper-evident.

**Controls on `main`:**

1. **No direct pushes** — pushes only via PR. `Restrict who can push to matching branches` set to no one (not even admins, with `Do not allow bypassing` enabled).
2. **Linear history required** — no merge commits. Forces squash- or rebase-merge, making each `main` commit a single reviewable, revertable unit.
3. **Required reviews** — minimum 1 for low-risk paths, 2 for sensitive paths via CODEOWNERS. `Dismiss stale reviews` on new commits so a reviewer's approval doesn't carry over after the author force-pushes.
4. **CODEOWNERS** mapping critical paths (`/infra/`, `/auth/`, `/billing/`, `/.github/workflows/`) to specific owners. Workflows-on-workflows is a common attack vector — protect `.github/` aggressively.
5. **Required status checks** — CI must pass: unit tests, integration tests, SAST (e.g. Semgrep), dependency scan (Dependabot/Snyk/Trivy), secret scan (gitleaks), license scan, and the duplicate-Q&A or contract-tests if relevant. Pin the exact check names so they can't be renamed away.
6. **Signed commits required** — see prior answer.
7. **Conversation resolution required** — every PR comment must be resolved before merge.
8. **No bypass for admins** — auditors will ask. The tradeoff: a real emergency (rare) requires temporarily lifting the rule, which is logged in the audit trail and is itself a finding to triage.

**Process controls layered on top:**

- **Ticket linking** — PR title regex enforced to include `JIRA-1234` (or similar). The `commit-msg` hook on the server rejects PRs without it. Auditors get traceability from ticket → PR → commit → deployment.
- **PR templates** — checkboxes for "tests added," "security impact considered," "rollback plan." Required for sensitive areas via CODEOWNERS-driven PR templates.
- **Production deploy gating** — `main` is continuously deployed to staging, but production promotion goes through a separate change-management workflow (ServiceNow, Linear, etc.) with the deploying engineer different from the commit author. Many auditors require this segregation of duties.
- **Audit log retention** — Git host audit logs (GitHub Enterprise, GitLab Premium) retained 1+ year and shipped to your SIEM. The audit log captures every protection-rule change, force-push attempt, and admin override.

**Where this hits ergonomics:**

- Two reviewers required on `/auth/` slows urgent fixes — mitigate with a documented break-glass process and clear on-call ownership.
- Required signed commits adds setup friction for new hires — build it into the laptop bootstrap script and have the onboarding checklist verify a signed test commit before they get repo write access.
- Many required checks → slow merges. Invest in CI parallelism and selective testing (run only the affected service's tests when paths under `services/foo/` change). Without this, developers will route around the system, defeating the policy.
- "No bypass for admins" feels paranoid until you're sitting in a SOC 2 audit; lean into it and design the emergency process upfront so engineers know what to do without disabling controls.

The principle: **make the secure path the easy path.** If the protected workflow is faster than circumventing it (because of good tooling, fast CI, and sane PR sizes), engineers stay on it; the policy then enforces itself culturally rather than only via lockdown.

---
