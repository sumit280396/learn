# Git Interview Guide — From Zero to Confident Answers
### Understand how Git actually works, then the answers write themselves

---

## PART 0: The Mental Model (10 minutes here saves you an hour of memorizing)

**What Git is:** a version control system — a time machine plus a collaboration protocol for files. Every meaningful state of your project is saved as a **commit**, you can return to any of them, and multiple people can work in parallel without overwriting each other.

**The single most important idea — Git stores SNAPSHOTS, and commits form a chain:**
Each commit is a full snapshot of the project at a moment, plus metadata: author, timestamp, message, and — crucially — **a pointer to its parent commit**. That parent-pointer chain is the history. A **branch** is nothing more than a movable name tag pointing at one commit. When you make a new commit on a branch, the tag slides forward. This is why branches in Git are instant and free — you're creating a 41-byte pointer, not copying the project.

**The three areas — where your changes live (memorize this trio):**
1. **Working directory** — the actual files you're editing. Just files. Git is watching, not saving.
2. **Staging area (index)** — the loading dock. `git add` places changes here; it's you composing *what the next commit will contain*. This is why staging exists at all: you can change five files but commit them as two clean, logical commits.
3. **Repository (.git)** — the permanent record. `git commit` takes what's staged and writes the snapshot.

> **The flow:** edit → `git add` (working → staging) → `git commit` (staging → repository) → `git push` (your repository → the shared remote).

**Local vs remote — the part that confuses everyone at first:**
Git is *distributed*: your laptop has the ENTIRE repository — full history, all branches. GitHub/GitLab/Bitbucket just host another copy everyone agrees to treat as the shared source of truth (the **remote**, conventionally named `origin`). `push` sends your commits up; `fetch` downloads new remote commits without touching your work; `pull` = fetch + merge them into your branch. Nothing you do locally affects anyone until you push.

**The daily workflow in six commands:**
```
git clone <url>        # copy the remote repo to your machine (once)
git checkout -b fix-x  # create + switch to a new branch
# ...edit files...
git add file.py        # stage
git commit -m "Fix X"  # snapshot
git push origin fix-x  # publish your branch
# then open a Merge Request / Pull Request for review
```

---

## PART 1: Core Concept Questions & Answers

**Q1. What is Git and why is it the standard?**
> "A distributed version control system: every clone holds the full history, commits are immutable snapshots linked into a graph, and branching is a constant-time pointer operation. That design gives you three things teams can't live without: complete history and traceability of every change (who, what, when, why — the commit message), safe parallel work through cheap branches, and offline capability since the whole repo is local. Traceability is also why Git underpins change control in regulated environments — the merge request *is* the reviewable, evidenced change."

**Q2. Explain `git add` vs `git commit` vs `git push`. (the fundamentals check)**
> "Three moves through three areas: `add` stages selected changes from the working directory into the index — composing the next commit; `commit` writes the staged snapshot permanently into the local repository with a message and author; `push` uploads local commits to the shared remote so others can see them. The separation matters: staging lets me turn a messy afternoon of edits into clean, logical, reviewable commits, and local-vs-push means I can commit freely — even half-done work on my branch — without affecting anyone."

**Q3. What is a branch, really? And HEAD?**
> "A branch is a movable pointer to a commit — nothing more; creating one copies nothing. **HEAD** is the pointer to *where I currently am* — normally it points at a branch name, and committing moves that branch forward. This model explains everything else: switching branches just moves HEAD and updates the working files to that snapshot; **detached HEAD** means HEAD points directly at a commit instead of a branch — you're sightseeing in history, and commits made there belong to no branch (recoverable, but confusing — see troubleshooting T3)."

**Q4. Merge vs rebase — the guaranteed question. Explain both and when to use which.**
> "Both integrate one branch's work into another; they differ in the history they leave.
> **Merge** (`git merge feature` while on main) creates a *merge commit* with two parents, preserving history exactly as it happened — truthful but potentially tangled with many merge bubbles.
> **Rebase** (`git rebase main` while on feature) *replays* my feature commits one-by-one on top of main's latest commit, as if I had started my work today — producing a clean, linear history, at the cost of rewriting my commits (new hashes).
> The iron rule: **never rebase commits that others already have** — rewriting shared history forces everyone else into conflict hell. Safe pattern, and my default: rebase my *private* feature branch onto main to stay current and keep history clean, then merge into main via a reviewed merge request. One-liner: *merge preserves truth, rebase rewrites for clarity; rewrite only what's yours alone.*"

**Q5. What is a merge conflict and how do you resolve one?**
> "A conflict happens when the same lines were changed differently on both branches — Git can auto-merge changes to *different* parts of files, but it won't guess between competing edits to the same lines. Git pauses the merge and marks the file:
> ```
> <<<<<<< HEAD
> your version
> =======
> their version
> >>>>>>> feature
> ```
> Resolution: open the file, decide — keep one side, the other, or hand-craft a combination — delete the markers, `git add` the resolved file, and complete the merge (`git commit`, or `git rebase --continue` mid-rebase). If it's a mess, `git merge --abort` returns to the pre-merge state safely. Prevention beats cure: small short-lived branches, pull/rebase frequently, and talk to whoever owns the other change when the conflict is semantic, not textual — Git resolves text; humans must resolve *intent*."

**Q6. `git fetch` vs `git pull`?**
> "`fetch` downloads new commits from the remote and updates my remote-tracking refs — my view of the remote — **without touching my working branch**; I can then inspect what changed before integrating. `pull` is fetch **plus** an immediate merge (or rebase, with `pull --rebase`) into my current branch. Fetch-then-look is the cautious habit; `pull --rebase` keeps a linear local history and is a common team default."

**Q7. `git reset` vs `git revert` vs `git checkout` — undoing things. (high-frequency question)**
> "Three different undos for three situations:
> **`revert`** — the *safe, public* undo: creates a **new commit** that applies the inverse of a bad commit. History is preserved, nothing rewritten — the only correct way to undo something already pushed/shared. This is also the IaC rollback story: revert + pipeline redeploy.
> **`reset`** — the *private* undo: moves the branch pointer backward. `--soft` keeps changes staged, `--mixed` (default) keeps them in the working directory, `--hard` discards them entirely (dangerous — work gone from the working tree). Only for commits that never left my machine.
> **`checkout`/`restore`** — discard uncommitted edits to a file (`git restore file.py`) or move around history.
> Decision rule to say out loud: *shared history → revert; local-only → reset; uncommitted → restore.*"

**Q8. What is `git stash`?**
> "A clipboard for uncommitted work: `git stash` shelves my in-progress changes and cleans the working directory — for when an urgent fix interrupts me mid-task and I need to switch branches without committing half-done work. `git stash pop` brings it back. Useful, but a stash graveyard is a smell — usually committing to a WIP branch is more durable and visible."

**Q9. What is `.gitignore` and what should never be in a repo?**
> "A file listing patterns Git should never track: build outputs, dependency directories (`node_modules`), editor junk, local configs — and above all **secrets and large data**. Secrets: because history is forever — a password committed once lives in every clone even after 'deleting' it (see T5). Large data: Git stores history of everything, so repos bloat permanently; data belongs in object storage or, if versioned files are truly needed, **Git LFS**, which stores pointers in Git and content elsewhere. In our Domino world this is a daily user conversation: code in Git, data in Datasets, secrets in the secret store — never crossed."

**Q10. Describe the branching workflow you'd use / Git workflows you know.**
> "**Feature-branch workflow** (my default and what GitLab/GitHub push you toward): main is protected and always releasable; every change is a short-lived branch → merge request → review + CI green → merge. **Trunk-based development**: even shorter-lived branches or direct-to-trunk with feature flags, optimized for continuous deployment — what the DORA research associates with elite performers. **GitFlow**: heavyweight — long-lived develop/release/hotfix branches; suits versioned, scheduled releases but adds ceremony most teams no longer need. The senior point: workflow follows release model — continuous delivery wants trunk-ish and short-lived branches; boxed releases can justify GitFlow."

**Q11. What is a tag?**
> "A permanent, immutable label on a specific commit — `v2.1.0` — unlike a branch, it doesn't move. Tags mark releases; annotated tags carry a message/author and are the standard for release points. In pipelines, tags are common deploy triggers, and in module ecosystems (Terraform modules!) version tags are what consumers pin to."

**Q12. What's actually inside `.git`? (depth-check question)**
> "An object database: **blobs** (file contents), **trees** (directory listings pointing to blobs), **commits** (pointing to one tree + parent commits + metadata), all addressed by SHA hashes — plus refs (branches/tags are just files containing a commit hash) and HEAD. Two nice consequences to mention: identical content is stored once regardless of filename (dedup by content hash), and history is tamper-evident — change anything in an old commit and every descendant hash changes, which is a quiet integrity property auditors appreciate."

---

## PART 2: Troubleshooting Questions & Answers

**T1. "I committed to the wrong branch (e.g., directly to main locally). Fix it."**
> "If it hasn't been pushed: copy the commit to the right place, then remove it from the wrong one. `git checkout correct-branch && git cherry-pick <commit-hash>` — cherry-pick applies any single commit onto the current branch — then back on main, `git reset --hard HEAD~1` to drop it. If it **was pushed** to a shared branch: no resets — `git revert` it on main (safe, public undo) and cherry-pick the original onto the right branch. The branch-protection moral: this shouldn't be possible on main at all — protected branches exist precisely because everyone does this once."

**T2. "I need to undo the last commit but keep my changes."**
> "`git reset --soft HEAD~1` — branch pointer steps back one, changes stay staged, ready to recommit properly (fix the message, split it, add the forgotten file). Related favorite: `git commit --amend` folds new staged changes into the last commit or edits its message. Both rewrite the last commit — fine locally, forbidden once pushed."

**T3. "Git says 'detached HEAD' — what happened and what do I do?"**
> "I checked out a specific commit (or tag) rather than a branch, so HEAD points at a commit directly — I can look around and even commit, but those commits hang off no branch and will look 'lost' when I leave. Fix: if I made commits I want, `git checkout -b rescue-branch` right there — a branch now points at them, saved. If just browsing, `git checkout main` returns to normal. Not an error — just Git telling you you're off the named path."

**T4. "I did `git reset --hard` / deleted a branch and lost commits. Recoverable?"**
> "Almost always yes — **`git reflog`** is the safety net most people don't know: a local journal of every position HEAD has occupied, even through resets and rebases. Find the lost commit's hash in the reflog, then `git checkout -b recovered <hash>` or `git reset --hard <hash>` to restore. Committed work is very hard to truly lose for ~90 days (reflog expiry). What reflog can NOT recover: changes that were never committed — `reset --hard` on uncommitted work is genuinely gone, which is why the command deserves respect."

**T5. "Someone committed a password/API key to the repo. Walk me through the response. (the security scenario — nail this)**
> "Order matters: **(1) Rotate the credential immediately** — that's the actual incident response; the secret must be treated as compromised the moment it was pushed, because deleting it later doesn't un-expose it — clones, forks, and CI caches already have it. **(2) Then clean history** — removing the file in a new commit is NOT enough; it lives in every prior commit. History rewrite with `git filter-repo` (or BFG Repo-Cleaner) purges it from all commits, then a coordinated force-push and everyone re-clones — disruptive, which is why step 1 is the real fix and step 2 is hygiene. **(3) Prevent recurrence** — secret-detection scanning in the pipeline (GitLab has it built in) and pre-commit hooks, so the next attempt is blocked at commit time. Answering rotation-first is what separates people who've handled this from people who've read about it."

**T6. "`git push` rejected: 'non-fast-forward / fetch first'. What's happening?"**
> "The remote branch has commits I don't have — someone pushed since I last pulled; Git refuses to overwrite them. Correct fix: `git pull --rebase` (replay my commits on top of theirs), resolve any conflicts, push again. The WRONG reflex is `git push --force`, which would obliterate their commits. If force is ever truly needed (after an agreed history rewrite), **`--force-with-lease`** is the safety catch — it fails if the remote moved since I last fetched, so I can't blindly destroy work I haven't seen."

**T7. "Someone force-pushed over the shared branch and commits vanished. Recover."**
> "The commits still exist — force-push moved the branch pointer, it didn't delete objects. Recovery sources, in order: anyone with the commits still local can simply push them back; the server-side reflog (GitLab/GitHub keep one; GitLab also shows deleted commit ranges in MR/push events); or CI runners that checked out those commits. Restore the branch to the correct hash, then the process fix: **protected branches with force-push disabled** — this incident is the standard justification for that setting, and after this it should be impossible, not discouraged."

**T8. "The repo has become huge / clone takes forever. Causes and fixes?"**
> "Someone committed large binaries or data at some point — and Git history keeps them forever even if later 'deleted'. Diagnose with `git count-objects -vH` and tools that list biggest objects in history. Fixes: for the future — `.gitignore`, Git LFS for legitimately large versioned assets, and repo policies/hooks blocking large files; for the past — history rewrite (`git filter-repo`) to purge the blobs, with the same coordination cost as T5. Workaround for consumers meanwhile: **shallow clone** (`git clone --depth 1`) — CI systems do this by default for speed. In a data science platform, this is a recurring user education topic: the answer to 'my repo is 4 GB' is 'your data belongs in Datasets, not Git.'"

---

## PART 3: Scenario / Design Questions

**S1. "How would you set up Git standards for our platform team?"**
> "Protected main — no direct pushes, no force-push; merge requests mandatory with at least one approval and CODEOWNERS for sensitive paths (Terraform, pipeline definitions); CI must be green to merge; short-lived feature branches with a naming convention tied to ticket IDs so every change traces to a change/incident record; commit messages that say *why*, not just what; secret-detection and large-file blocking in the pipeline; version tags for releases and module versions. None of this is bureaucracy — it's what makes `git log` a usable audit trail, which in our GxP context is literally compliance evidence."

**S2. "A data scientist says 'Git rejected my push and now everything is broken.' They're panicked. Handle it."**
> "First the human part: almost nothing in Git is truly lost — committed work is recoverable, so lower the temperature before touching the keyboard. Then diagnose from `git status` and the exact error: nine times out of ten it's the non-fast-forward rejection (T6) → `pull --rebase`, help them through any conflict, push. If they're mid-merge with scary markers in files, `git merge --abort` resets to safety and we retry calmly. Then the platform lesson: most scientist Git pain follows the same three patterns — push rejection, conflicts on shared notebooks, and committing data — so I'd fold the fixes into onboarding docs and a cheat sheet rather than re-answering the ticket forever. Support that teaches beats support that resolves."

**S3. "Two people edited the same Jupyter notebook and the merge is garbage. Why, and what do you recommend?"**
> "Notebooks are JSON containing code *plus* outputs and execution counters — so two runs of the same notebook differ even with identical code, and line-based merge produces nonsense. Mitigations: strip outputs before committing (pre-commit hook — `nbstripout` — makes diffs meaningful), notebook-aware diff/merge tooling (`nbdime`, or GitLab/Domino's rendered notebook diffs), and structurally: keep shared logic in `.py` modules that merge cleanly, with notebooks as thin exploration layers, and avoid two people editing one notebook simultaneously — split work by file. This question is really testing whether I understand my *users*, and this is the single most common Git complaint on a data science platform."

**S4. "How does Git connect to everything else in this role?"**
> "Git is the spine the rest attaches to: **Domino** — Git-based projects sync repos into workspaces; I'd support the service credentials/deploy keys and SSH-vs-HTTPS auth issues that generates. **Terraform** — the repo is the desired state; `git revert` is the rollback plan; tags version modules. **CI/CD** — every push/MR triggers pipelines; protected branches gate deploys; the MR with its green pipeline is the change record's evidence. **Change management** — branch = change in progress, MR = review and approval, merge = authorized implementation, history = audit trail. One tool, four of this JD's bullet points."

---

## PART 4: Rapid-Fire One-Liners (skim before the interview)

- **Three areas:** working directory → (`add`) → staging/index → (`commit`) → repository → (`push`) → remote
- **Branch** = movable pointer to a commit; **HEAD** = where you are; **tag** = immutable pointer
- **fetch** = download only; **pull** = fetch + merge; **pull --rebase** = fetch + replay (linear)
- **Merge** preserves history (merge commit, two parents); **rebase** rewrites for linearity — never rebase shared commits
- **Undo:** shared → `revert` (new inverse commit); local → `reset` (soft=keep staged / mixed=keep files / hard=discard); uncommitted → `restore`
- **`commit --amend`** = fix the last commit (local only)
- **`cherry-pick <hash>`** = copy one commit onto current branch
- **`stash`** = shelf for uncommitted work
- **`reflog`** = journal of HEAD positions — the recovery tool for "lost" commits (~90 days)
- **Conflict markers:** `<<<<<<< HEAD` yours / `=======` / `>>>>>>>` theirs — resolve, `add`, continue; `--abort` bails safely
- **Committed secret:** rotate FIRST, then `git filter-repo` history purge, then secret-scanning to prevent
- **Rejected push (non-fast-forward)** = remote moved → `pull --rebase`; never bare `--force`; if rewrite agreed, `--force-with-lease`
- **Big repo** = binaries in history → LFS/`.gitignore`/filter-repo; CI uses shallow clones (`--depth 1`)
- **Notebooks merge badly** because they're JSON with outputs → `nbstripout`, `nbdime`, logic in `.py` files
- **Workflows:** feature-branch + protected main (default) / trunk-based (CD-optimized) / GitFlow (versioned releases, heavyweight)

---

## FINAL RECALL STORY (read 3 times — chains every concept)

> *I branch off protected **main** — a new pointer, instant — and work: edit, **add** to staging, **commit** snapshots with real messages. Main moves on beneath me, so I **rebase my private branch** onto it for a clean linear history — safe, nobody has my commits. One file conflicts: markers show **HEAD's version vs mine**; I resolve intent, `add`, continue. My **push is rejected** — a teammate pushed first — `pull --rebase`, push again; no `--force`, ever, on shared branches. The **merge request** runs CI, gets an approval, merges — that MR is the change record. Next day a bad commit is live: it's shared history, so **revert**, not reset — a new inverse commit, deployed by the pipeline. A panicked scientist "lost everything" after `reset --hard`: **reflog** finds the hash, a rescue branch saves it — committed work is nearly immortal. Then the real incident: an API key pushed to the repo — **rotate first**, purge history with **filter-repo** second, turn on **secret detection** third. And the 4 GB repo that takes ten minutes to clone? Data committed in March — moved to Datasets, history filtered, **LFS and size hooks** so it can't recur. The audit later walks our `git log` end to end and finds every change reviewed, evidenced, attributable — which is why all these habits were never bureaucracy.*

Retell that story and there is no Git interview question that will surprise you.
