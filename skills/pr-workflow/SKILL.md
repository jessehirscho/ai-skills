---
name: pr-workflow
description: Use whenever creating, updating, or working with a GitHub pull request via the `gh` CLI — opening a PR, reading or replying to review comments, checking CI status, or handling merge conflicts on a PR branch. Covers the mechanics of `gh`; for PR policy (test-plan format, never merging to main) see workflow-conventions.
---

# PR Workflow (gh CLI mechanics)

Operational companion to `workflow-conventions`. That skill covers *policy* (never merge, test-plan checklist format, session handoffs) — this skill covers the *mechanics* of driving a PR through `gh`. Don't repeat the policy content here; just follow it.

## Pre-PR checklist

```bash
git branch --show-current        # confirm NOT main/master
git status                       # check for stray/untracked files
git diff                         # review unstaged changes
git diff --staged                # review staged changes
git log origin/<branch>..HEAD    # confirm commits pushed (empty output = all pushed)
git push -u origin <branch>      # push if needed
```

If `git status` shows anything unexpected, stop and look before proceeding — never blindly `add -A`.

## Creating a PR

Always use a heredoc for the body so markdown (headers, checklists) renders correctly — a plain `--body "..."` string mangles newlines.

```bash
gh pr create --title "Fix booking form validation on mobile" --body "$(cat <<'EOF'
## Summary
- What changed and why

## Test plan
- [ ] Specific thing to check on desktop
- [ ] Specific thing to check on mobile (375px)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

The Summary + Test plan format is required by `workflow-conventions` — this skill just supplies the `gh` incantation to submit it correctly.

Draft PR: add `--draft`. Target a non-default base: add `--base <branch>`.

## Reading review comments

```bash
gh pr view <number> --comments              # general PR-level comments + review summaries
gh pr view <number> --json reviews          # structured review data (approved/changes-requested/state)
gh api repos/{owner}/{repo}/pulls/<number>/comments   # inline code review comments (line-anchored)
gh api repos/{owner}/{repo}/pulls/<number>/reviews    # top-level review objects
```

- `pulls/<number>/comments` = inline comments attached to a specific diff line/file.
- `pulls/<number>/reviews` = the review itself (APPROVED / CHANGES_REQUESTED / COMMENTED) plus its summary body.
- `issues/<number>/comments` = general PR conversation comments (a PR is also an issue in the API).

To find `{owner}/{repo}`: `gh repo view --json owner,name -q '.owner.login + "/" + .name'`.

## Checking CI / checks status

```bash
gh pr checks <number>                       # pass/fail summary for all checks
gh pr checks <number> --watch               # poll until checks finish
gh run list --branch <branch> --limit 5     # recent workflow runs for this branch
gh run view <run-id> --log-failed           # logs for just the failed steps (fast triage)
gh run view <run-id> --log                  # full logs if --log-failed isn't enough
```

Diagnose from `--log-failed` before re-running anything. If a rerun is warranted: `gh run rerun <run-id> --failed`.

## Updating a PR after review feedback

- Push new commits addressing feedback — do **not** amend/force-push over commits that are already published on the PR unless the user explicitly asks for that history rewrite.
  ```bash
  git add <files>
  git commit -m "Address review feedback: <what>"
  git push
  ```
- Reply to a specific inline review comment thread (keeps it threaded, unlike a new general comment):
  ```bash
  gh api repos/{owner}/{repo}/pulls/<number>/comments \
    -f body="Fixed in <sha> — moved validation to onBlur." \
    -F in_reply_to=<comment_id>
  ```
- Mark a review thread resolved (GraphQL — `gh api` REST has no endpoint for this):
  ```bash
  gh api graphql -f query='
    mutation { resolveReviewThread(input: {threadId: "<thread_node_id>"}) { thread { isResolved } } }'
  ```
  Get `thread_node_id` values via:
  ```bash
  gh api graphql -f query='
    query { repository(owner:"{owner}", name:"{repo}") { pullRequest(number: <number>) {
      reviewThreads(first: 50) { nodes { id isResolved comments(first:1) { nodes { body } } } } } } }'
  ```

## Quick reference

| Task | Command |
|---|---|
| View PR in browser | `gh pr view <number> --web` |
| View PR diff | `gh pr diff <number>` |
| List open PRs | `gh pr list` |
| Check status/checks | `gh pr checks <number>` |
| Add a general comment | `gh pr comment <number> --body "..."` |
| Edit title/body | `gh pr edit <number> --title "..." --body "..."` |
| Mark draft ready | `gh pr ready <number>` |
| Convert to draft | no direct `gh pr` command to undo "ready"; close and reopen as draft if needed (`gh pr close <number>` then `gh pr reopen <number> --draft` if your `gh` version supports it) |
| Close without merging | `gh pr close <number>` |
| Reopen a closed PR | `gh pr reopen <number>` |
| Checkout a PR locally | `gh pr checkout <number>` |

## Handling merge conflicts on a PR branch

```bash
git fetch origin
git status                       # confirm nothing uncommitted before rebasing
git rebase origin/main           # or: git merge origin/main, per repo convention
# resolve conflicts in editor, then:
git add <resolved files>
git rebase --continue
git push --force-with-lease      # required after rebase rewrites history — only for YOUR branch
```

Use `--force-with-lease`, never bare `--force` — it aborts if the remote has commits you haven't seen (e.g. someone else pushed to your PR branch). Confirm with the user before force-pushing if there's any chance of shared work on the branch.

## What NOT to do

- Never run `gh pr merge` or otherwise merge a PR — merging is the human's job (see `workflow-conventions`).
- Never force-push over a branch without explicit ask, and never use bare `--force` when `--force-with-lease` will do.
- Never skip or bypass a failing CI check to get a PR mergeable — fix the underlying failure or flag it to the user.
- Never amend/rewrite commits already pushed to a PR branch that others may be reviewing, without asking first.
