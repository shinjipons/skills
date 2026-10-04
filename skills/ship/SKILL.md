---
name: ship
description: >
  Commit, push, open a pull request, wait for its CI checks, and merge it with
  the branch deleted on both sides once they pass. Only when the user types
  /ship; typing it is the authorization to merge.
disable-model-invocation: true
argument-hint: "[branch name]"
---

# Ship

The user typed `/ship`. That is their standing authorization, for this one
branch, to merge the pull request without asking again once CI is green. It
does not authorize merging past a failing or cancelled check, a merge conflict,
or a requested-changes review.

Branch name: `$ARGUMENTS` if given; otherwise choose a descriptive one, or keep
the current branch if it is not the base.

Base branch: always the repository's default branch
(`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`).

## 1. Get onto a branch

- If a branch name was given and it is not the current branch, create it with
  `git switch -c <name>`, taking the uncommitted work with it. If a branch of
  that name already exists, locally or on origin, stop and ask the user rather
  than reuse or overwrite it.
- If no name was given and the current branch is the base branch, create a
  descriptive branch and move the work onto it. Never push straight to the base.
- Name the branch well now. Renaming it after the PR is open closes the PR.

## 2. Commit and push

- If there are uncommitted changes, look at the diff, stage it, and commit with
  a message that says what changed and why. Follow any attribution lines the
  session asks for.
- Push with `git push -u origin <branch>`.

## 3. Open the pull request, or reuse it

- If `gh pr list --head <branch> --state open` finds one, reuse it and skip to
  step 4.
- Otherwise run `gh pr create --base <base>`, not as a draft. Write a title that
  summarizes the change, and a body covering what changed, why, and what a
  reviewer should know. Follow the repo's PR template if it has one.
- Report the URL.

## 4. Wait for CI

- Run `gh pr checks <n> --watch --fail-fast` in the background and wait for it
  to finish.
- If the PR has no checks at all, wait about 30 seconds and look again, since
  checks can register late. If there are still none, treat that as passing.
- If any check fails or is cancelled, stop. Report the failing check names and
  the shortest decisive log line. Do not merge.

## 5. Merge

- Confirm `gh pr view <n> --json mergeable,reviewDecision` shows it mergeable
  and not `CHANGES_REQUESTED`. If not, stop and report why.
- In the Claude desktop app, read `mcp__ccd_pr__get_status` first and note the
  bound URL.
- Merge with `gh pr merge <n> --merge --delete-branch`. This deletes the remote
  and local branch only after the merge succeeds.

## 6. Clean up

- Switch to the base branch and `git pull --ff-only`.
- Run `git fetch --prune`. Confirm the branch is gone with
  `git ls-remote --heads origin <branch>`, and locally with `git branch`.
- If a local copy survived, delete it with `git branch -d <branch>`. Never use
  `-D`; if `-d` refuses, report it.
- In the desktop app, dismiss the PR with `mcp__ccd_pr__unbind_pr` and the URL
  you noted, then close the `pr` pane if `mcp__ccd_view__get_layout` shows it
  open.

## 7. Report

In a few lines: the PR URL, the merge commit, and that the branch was deleted on
both sides. Or, if the merge did not happen, the step that stopped it and why.
