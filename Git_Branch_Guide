# Git Branch Workflow Guide

Use this guide to clone a repository, set up a local branch, work safely, and prepare changes for review. Replace every `<placeholder>` with the real value and omit the angle brackets. For example, replace `<branch-name>` with `feature/login`; do not type `<` or `>`. The examples use `origin` as the remote name and `main` as the base branch; use your repository's actual remote and base branch names when they differ.

## 1. Clone the repository

Get the repository URL from your company or repository hosting page. Choose either HTTPS or SSH, depending on your organization's access setup:

```bash
git clone https://github.com/<organization>/<repository>.git
```

```bash
git clone git@github.com:<organization>/<repository>.git
```

Replace `<organization>` and `<repository>` with the owner or organization name and repository name. HTTPS may require signing in or a credential manager; SSH requires an SSH key that is registered with the host. Your organization may use a different host name or URL format.

Change into the directory created by `git clone`, then verify the remote and working-tree status:

```bash
cd <repository>
git remote -v
git status --short --branch
```

If you want a different local directory name, provide it as the final argument to `git clone <url> <directory-name>`, then use that directory with `cd`. `git remote -v` should show the clone URL under `origin`. If the remote has another name, use that name in place of `origin` below.

## 2. Set up a branch

### If your company already created a remote branch

Fetch the latest remote branch information, then create a local branch that tracks the supplied remote branch:

```bash
git fetch --prune origin
git switch --track origin/<branch-name>
```

`git switch --track` creates a local branch and connects it to the remote-tracking branch. If a local branch with that name already exists, switch to it instead of creating it again, and check its tracking relationship:

```bash
git switch <branch-name>
git branch -vv
```

In `git branch -vv`, an entry such as `[origin/<branch-name>]` indicates the configured upstream. If the local branch has no upstream, set it explicitly:

```bash
git branch --set-upstream-to=origin/<branch-name> <branch-name>
```

### If you are creating a new branch

Confirm the base branch and naming rules with your team first. Check that you have no uncommitted changes to lose, fetch the latest remote state, update your local base branch using fast-forward only, and create your working branch:

```bash
git status --short
git fetch origin
git switch main
git pull --ff-only origin main
git switch -c feature/<short-task-name>
git push -u origin feature/<short-task-name>
```

Replace `main` if your team uses another base branch, and replace `<short-task-name>` with a concise description, such as `feature/login`. `git pull --ff-only` advances the local base branch only when it can do so without creating a merge commit; it stops if the histories have diverged. If you have uncommitted work, commit it or save it first, for example with `git stash push -m "<work description>"`; restore it later with `git stash pop`. Do not use reset, clean, or a forced branch switch without first confirming what would be discarded.

`git push -u` publishes the new branch and sets its upstream. After that, `git push` and `git pull` usually target the configured upstream automatically.

## 3. Switch branches and check status

```bash
git branch --show-current
git status --short --branch
git branch -vv
git fetch origin
git status -sb
git log --oneline --decorate --graph -10
```

`git status` shows the current branch and tracked, modified, and untracked files. After `git fetch`, `ahead` and `behind` in `git status -sb` compare your local branch with its configured remote-tracking branch. This comparison is only as current as the last fetch. Fetch updates remote-tracking information but does not merge changes into your current branch or update its working files.

Before switching branches, commit, stash, or otherwise preserve your current changes. Git may allow some changes to remain during a switch, but that depends on whether they conflict with the target branch; do not rely on a switch as a way to save work.

```bash
git switch <branch-name>
```

## 4. Review changes, commit, and push

Inspect your changes and stage only the files you intend to include:

```bash
git status
git diff
git diff --check
git add <file-path>
git diff --cached
git commit -m "<short change summary>"
git status --short --branch
```

Use `git add -A` only when you have reviewed and intend to stage all changes. Before committing, make sure you have not staged secrets such as tokens, passwords, or private configuration.

```bash
git push -u origin <branch-name>  # First push or when no upstream is configured
git push                          # When an upstream is already configured
```

Do not push directly to a protected or shared base branch such as `main` or `develop`. Push your working branch and use your team's pull request (PR) or merge request (MR) review process. `git push --force` can overwrite remote commits; do not use it on shared branches. Even on a personal branch, check team policy first. If rewriting its history is necessary and permitted, prefer `--force-with-lease` over plain `--force`.

## 5. Update from the base branch and merge

To bring the latest base-branch changes into your working branch, fetch first and then switch to your branch:

```bash
git fetch origin
git switch <branch-name>
```

If your team uses merge commits to update working branches:

```bash
git merge origin/main
```

If your team permits rebasing and nobody else is relying on your branch, you can instead replay your commits on top of the base branch:

```bash
git rebase origin/main
```

If conflicts occur, resolve and inspect the affected files. During a merge, stage the resolved files with `git add <file-path>` and complete the merge with `git commit`. During a rebase, stage the resolved files and continue with `git rebase --continue`. To stop and return to the pre-operation state, use `git merge --abort` or `git rebase --abort`, respectively; check your working-tree state before aborting.

After review and required checks, merge through the PR/MR process approved by your team. Do not use a local merge to bypass branch protection or required reviews. Merge permissions, approvals, and available merge methods depend on repository settings and team policy.

## 6. Roll back or undo changes

The right rollback depends on whether the change is unstaged, committed locally, or already published. Before acting, confirm your current branch and target commit with `git status` and `git log --oneline --decorate -5`. Commit or stash anything you need to keep.

### Uncommitted changes

Unstage a file while keeping its edits:

```bash
git restore --staged <file-path>
```

Discard uncommitted edits to a file:

```bash
git restore <file-path>
```

The second command discards that file's uncommitted changes and may be difficult to undo. `git restore` does not remove untracked files. `git clean` can delete untracked files; preview its targets first with `git clean -n` and do not proceed unless you have confirmed they are safe to remove.

### Last commit that has not been published

Remove the last commit but keep its changes staged:

```bash
git reset --soft HEAD~1
```

Remove the last commit but keep its changes in the working tree, unstaged:

```bash
git reset HEAD~1
```

`git reset --hard HEAD~1` can discard both the commit and working-tree changes. Avoid it unless you have verified that everything affected is safe to lose or has been backed up.

### A commit that has already been pushed or shared

For a published commit, do not reset a shared branch and force-push to erase it. Create a new commit that reverses the change:

```bash
git log --oneline
git revert <commit-id>
git push
```

Reverting a merge commit requires selecting which parent to retain. Inspect its parents and agree on the intended result with your team; `-m 1` keeps the first parent:

```bash
git show --no-patch --pretty=raw <merge-commit-id>
git revert -m 1 <merge-commit-id>
git push
```

A revert remains in the project history, and reapplying the original change later may require additional work. Follow your team's PR/MR and deployment process when rolling back a shared or production change.

## 7. Keep work evidence

Record useful command output and change summaries in the relevant ticket or PR/MR so others can follow the work:

```bash
git status --short --branch
git branch -vv
git log --oneline --decorate -10
git diff --stat origin/main...HEAD
git diff --check
git show --stat --oneline HEAD
```

Include the base and working branch names, a change summary, links to relevant commits or PRs/MRs, checks run and their results, and any remaining risks or follow-up work. Review captured output before sharing it because it may contain secrets or sensitive file paths.

## 8. Code review checklist

- [ ] I confirmed the repository, working branch, and base branch.
- [ ] I checked `git status` and reviewed modified and untracked files; no unintended or sensitive files are included.
- [ ] I reviewed the full change with `git diff` or `git diff origin/main...HEAD`.
- [ ] `git diff --check` passes, and I ran the tests or checks required by this project.
- [ ] The commit message and PR/MR description explain the purpose, impact, and verification results.
- [ ] I did not push directly to a shared base branch and followed required review and approval steps.
- [ ] If a rollback was needed, I used `revert` for published commits and checked the impact.

Fetch the latest remote information before reviewing, and confirm that `main` is the correct base branch for this repository. The `...` comparison shows changes since the branches' common ancestor:

```bash
git fetch origin
git diff --check origin/main...HEAD
git diff --stat origin/main...HEAD
git diff origin/main...HEAD
```

PR/MR interfaces, required checks, review rules, and merge methods vary by repository. This guide describes general Git behavior and does not replace your organization's policies.

## 9. Set up GitHub MCP in Copilot

GitHub MCP lets Copilot use supported GitHub tools in an MCP-enabled client. Setup depends on the Copilot product, client, and organization policy; a repository can also provide MCP configuration that applies to that repository. Do not copy configuration examples from another client or invent a server URL, command, package name, or credential field: use the setup instructions for the exact client and server you intend to use.

1. Check whether GitHub MCP is already available in your Copilot client. If the repository or organization supplies an MCP server configuration, review its documented source and scope before enabling it.
2. If you need a personal server, open your Copilot client's MCP or installed-server settings, find the GitHub MCP server or add it using the client-specific official setup instructions, and select the GitHub account or host you intend to use.
3. Complete authentication in the client or its supported sign-in flow. Do not paste access tokens, passwords, or private keys into prompts, source files, commits, or this guide. Grant only the access needed for the requested work.
4. For repository-defined MCP servers, use the repository or project customization settings documented by that client. Inspect repository configuration changes before trusting or enabling a server, especially when they originate from an unfamiliar branch.
5. Confirm the server reports a successful connection and that the expected GitHub tools are available. If sign-in, permissions, or organization policy blocks access, use the client's supported settings or ask the repository administrator; do not work around the restriction with an unapproved token.

Global or personal server installation and repository-provided configuration are distinct: the former is generally managed in the user's client settings, while the latter is associated with a repository and may need explicit approval. Names, UI labels, configuration formats, and authentication steps differ by client version and host. Follow the current official instructions for your Copilot client; never commit personal credentials or assume that a repository configuration automatically authenticates you.

## 10. Request and work through a Copilot code review

Use code review as an additional set of eyes, not a substitute for understanding the change, running tests, or following required human review. Review a focused diff on your working branch and give Copilot enough context to assess intended behavior.

1. Make sure the branch is up to date with the intended base and that the diff contains only the work being reviewed. Identify the goal, relevant requirements, affected areas, expected behavior, and important constraints.
2. Ask Copilot to review the specific PR or diff for actionable bugs, regressions, missing edge cases, and test gaps. State the intended base branch and any behavior that must remain unchanged. Ask for findings with file/line references, impact, and a concise rationale; avoid asking for an unrestricted rewrite when you want a review.
3. Evaluate each finding against the actual code and requirements. Reproduce the issue or inspect the relevant call paths and tests; distinguish confirmed defects from suggestions, false positives, and intentional trade-offs. Do not apply a suggestion solely because it came from a review.
4. For accepted findings, make a focused fix on the working branch, add or update tests where appropriate, inspect the resulting diff, and run the project's relevant checks. For declined or deferred findings, record the reason in the PR discussion or review notes when useful.
5. Re-run review or targeted checks after substantial changes, then use the repository's normal PR/MR approval process. Do not treat Copilot review as an approval that satisfies required human reviewers or branch protection.

Keep the review scope bounded: a smaller, clearly described change is easier to validate than a broad mixed-purpose diff. If the branch changes while review is in progress, refresh the diff and confirm the findings still apply before acting.

## 11. Refactor safely

Refactoring changes code structure without intentionally changing externally observable behavior. Keep the work on a dedicated branch and make the result easy to compare and revert:

1. Define the behavior that must remain unchanged, the refactoring boundary, and any explicit non-goals. Check for existing tests, callers, and documented contracts; clarify ambiguous behavior before changing it.
2. Establish a baseline: inspect the working tree and relevant diff, run the focused tests and checks before editing, and record any existing failures so they are not mistaken for regressions.
3. Make one small, coherent structural change at a time. Preserve interfaces and behavior unless the task explicitly authorizes a behavior change. Avoid mixing formatting, dependency upgrades, and unrelated cleanup into the same refactor.
4. After each step, inspect the diff and run the most relevant tests. Once the refactor is complete, run the required broader tests, linters, type checks, or build. If a check fails, determine whether the refactor caused it rather than silently accepting or masking the failure.
5. Compare the final diff with the base, check for accidental files or behavior changes, and request a focused Copilot or human review using the guidance above. Commit the verified refactor with a clear message; push the working branch and use the normal PR/MR process.

If the change alters behavior, document that explicitly and add tests for the new behavior rather than presenting it as a behavior-preserving refactor. Keep rollback straightforward: small commits and a clean, reviewed diff are easier to revert than a large mixed-purpose change.
