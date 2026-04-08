---
name: create-stacked-diff-prs
description: Split a feature branch into a stack of incremental, reviewable PRs that merge into the feature branch. Use when a branch has grown too large for a single review, or when the user wants to break changes into smaller dependent PRs. Triggers on phrases like "stacked PRs", "stacked diffs", "split into PRs", "break up this branch".
license: MIT
argument-hint: "[base-branch (default: main)]"
metadata:
  author: widal001
  version: "2.0.0"
allowed-tools: Bash(git:*) Bash(gh:*) Read Grep Glob
---

# Create Stacked Diff PRs

Split the current feature branch into a series of small, dependent pull requests (a "stack") that merge into the feature branch, not directly into the base branch. A parent PR from the feature branch to the base branch ties everything together.

## Architecture

```
main <-- feature-branch (parent PR, closes the issue)
            ^
            |-- stack/feature/01-models   (PR targeting feature-branch)
            |      ^
            |      |-- stack/feature/02-api   (PR targeting 01-models)
            |             ^
            |             |-- stack/feature/03-tests   (PR targeting 02-api)
```

- **Stack PRs** are small, focused PRs that each merge into the feature branch via the stack chain.
- **Parent PR** goes from the feature branch to the base branch (e.g., `main`), links to the issue, and references all stack PRs.
- **Feature branch** serves as the accumulation point. It starts at the base branch (no commits ahead) and collects changes as stack PRs are squash-merged into it.
- Reviewers review each stack PR independently. After each merges (squash-merge into the feature branch), downstream PRs are rebased.

## Step-by-step process

### 1. Gather context

Determine the base branch. Use the argument `$ARGUMENTS` if provided, otherwise default to `main`.

Run the following to understand what's on the branch:

```
git log --oneline <base>..HEAD
git diff --stat <base>..HEAD
```

Also check for a PR template:

```
# Check common locations
cat .github/pull_request_template.md 2>/dev/null || \
cat .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null || \
cat docs/pull_request_template.md 2>/dev/null || \
echo "No PR template found"
```

Present a summary: number of commits, files changed, insertions/deletions.

### 2. Plan the commits (approval gate 1)

Before writing any code or creating branches, propose a **commit plan** that describes how the work will be organized into discrete, squash-ready commits. Each commit should:

- Represent a relatively isolated, stable unit of work
- Follow semantic or conventional commit structure (e.g., `docs: extract client docs to sub-package README`)
- Be independently valid (format, lint, and CI should pass after each commit)

Present the plan as a numbered list with:
- A conventional commit message for each commit
- Which files will be created, modified, or removed
- A one-line description of what the commit accomplishes

**Ask the user to confirm or adjust the commit plan before proceeding.** Do not write code until the user approves.

After approval, implement the commits. Run the project's format and lint checks before each commit to avoid needing fixup commits later. Scope format/lint/CI commands to the relevant package directory, not the workspace root, to avoid side effects on other packages.

If commits need to be reordered or squashed after the fact, use `git rebase` with `GIT_SEQUENCE_EDITOR` to do this non-interactively before creating stack branches.

### 3. Plan the stacked PRs (approval gate 2)

Once the commits are finalized, propose the **stacking plan**. Follow these principles:

- **Split by commit**: Each commit (or small group of tightly related commits) becomes one stack PR.
- **Keep the stack short**: Aim for 2-5 PRs, never more than 7. If there are many small commits, group related ones together.
- **Preserve commit order**: The stack should follow the original commit order, since later commits may depend on earlier ones.

Present the proposed stack as a numbered list with:
- Which commits go in each PR
- A short name for the PR (used in branch naming)
- A one-line description of what the PR accomplishes

**Ask the user to confirm or adjust the stacking plan before proceeding.** Do not create any branches or PRs until the user approves.

### 4. Derive a stack prefix

Derive a short, descriptive prefix from the current branch name. For example:
- Branch `feature/auth-system` produces prefix `auth-system`
- Branch `623/refactor-ts-sdk-readme` produces prefix `623-refactor-ts-sdk-readme`

Stack branches will be named: `stack/<prefix>/01-<slug>`, `stack/<prefix>/02-<slug>`, etc.

### 5. Prepare the feature branch

The feature branch must be at the same commit as the base branch so it can accumulate stack PRs as they merge. If the feature branch has commits ahead of the base:

1. Note the commit hashes you'll need for cherry-picking.
2. Reset the feature branch to the base branch:
   ```
   git checkout <feature-branch>
   git reset --hard <base>
   ```
3. Create a placeholder commit so GitHub allows creating the parent PR (GitHub requires at least one commit difference):
   ```
   git commit --allow-empty -m "chore: initialize feature branch for stacked diff"
   ```

### 6. Create the stack branches

For each PR in the stack, build branches incrementally by cherry-picking from the original commits:

1. Create the first stack branch from the base branch:
   ```
   git checkout <base>
   git checkout -b stack/<prefix>/01-<slug>
   git cherry-pick <commits-for-this-slice>
   ```
2. For subsequent branches, branch from the previous stack branch:
   ```
   git checkout stack/<prefix>/01-<slug>
   git checkout -b stack/<prefix>/02-<slug>
   git cherry-pick <commits-for-this-slice>
   ```
3. Repeat for all slices.

**Handle conflicts carefully**: if cherry-picking produces conflicts, stop and ask the user how to proceed.

After creating all stack branches, push everything:
```
git push -u origin <feature-branch>
git push -u origin stack/<prefix>/01-<slug>
git push -u origin stack/<prefix>/02-<slug>
git push -u origin stack/<prefix>/03-<slug>
```

Then return to the feature branch:
```
git checkout <feature-branch>
```

### 7. Create the parent PR

Create the parent PR **first** so its number can be included in the stack PRs. This PR goes from the feature branch to the base branch and:
- Closes the relevant issue (ask the user for the issue number if not obvious from the branch name)
- Lists all stack PRs

```
gh pr create \
  --base <base-branch> \
  --head <feature-branch> \
  --title "<description>" \
  --body "$(cat <<'EOF'
### Summary

- Fixes #<issue-number>
- Time to review: This PR is broken into a stack of smaller PRs for easier review.

### Changes proposed

- <Bulleted list of files added, changed, or removed across the full feature>

### Context for reviewers

This PR is the parent of a stacked diff. Please review the individual stack PRs listed below rather than reviewing this PR's cumulative diff directly.

### Stack PRs (review in order)

| # | PR | Description |
|---|-----|-------------|
| 1 | TBD | <pr1-description> |
| 2 | TBD | <pr2-description> |
| ... | ... | ... |

### Additional information

Each stack PR should be squash-merged into this feature branch in order. After the full stack is merged, this parent PR can be merged into `<base-branch>`.
EOF
)"
```

### 8. Create the stack PRs

For each stack branch, create a PR using `gh pr create`.

- The **first stack PR** targets the **feature branch** as its base.
- Each **subsequent stack PR** targets the **previous stack branch** as its base.
- **Title format**: `[Stack N/M] <conventional-commit-style description>`

For the PR body, use the repo's PR template (found in step 1) as the base structure, and **append a "Stack" section** at the end. If no PR template exists, use a simple summary format.

**PR description guidelines:**
- **Changes proposed**: A bulleted list of specific files, methods, or components added, changed, or removed. One bullet per change. Not a paragraph.
- **Context for reviewers**: Split into (1) **Review instructions** such as links to rendered preview files on the branch, and (2) **Implementation notes** covering tradeoffs, design decisions, or background context.
- **Additional information**: Reserved for screenshots, links to workflow runs, or PR previews showing the changes work. Use "N/A" if none.
- **Never use `---` (horizontal rules)** to separate sections. They look messy in GitHub.

Example:

```
gh pr create \
  --base <previous-branch-or-feature-branch> \
  --head stack/<prefix>/NN-<slug> \
  --title "[Stack N/M] <description>" \
  --body "$(cat <<'EOF'
### Summary

- Part N of M in a stacked diff
- Time to review: [xx] minutes

### Changes proposed

- Added `path/to/new-file` covering <what it covers>
- Updated `path/to/existing-file` to <what changed and why>
- Removed `path/to/old-file` because <reason>

### Context for reviewers

**Review instructions:**
- Review the [rendered file](https://github.com/<owner>/<repo>/blob/<branch>/path/to/file) for formatting and completeness

**Implementation notes:**
- <Tradeoff, design decision, or background context>

### Additional information

N/A

### Stack

> This PR is part of a stacked diff. Please review only the changes in this PR, not the cumulative diff.

| # | Branch | PR | Status |
|---|--------|----|--------|
| - | `<feature-branch>` (parent) | #<parent-pr> | 🏠 Base |
| 1 | `stack/<prefix>/01-<slug>` | #<number> | ✅ Current / ⏳ Pending |
| 2 | `stack/<prefix>/02-<slug>` | #<number> | ⏳ Pending |
| ... | ... | ... | ... |

**Review order**: Start from PR #1 and work down.
**Merge strategy**: Squash and merge each PR in order. After merging, rebase downstream PRs onto the updated feature branch if needed.
EOF
)"
```

### 9. Backfill PR numbers

After all PRs (stack + parent) are created, update each PR's body so the stack tables include all PR numbers. Use `gh pr edit` for this:

```
gh pr edit <pr-number> --body "<updated-body-with-all-pr-numbers>"
```

### 10. Report the result

Present the complete stack to the user:

- The parent PR number, title, and URL
- Each stack PR with its number, title, URL, and base branch
- Remind the user:
  - Stack PRs should be reviewed and merged in order (1 first, then 2, etc.)
  - Use squash-and-merge for each stack PR into the feature branch
  - After merging a stack PR, downstream PRs may need rebasing (especially if changes were made during review)
  - Once all stack PRs are merged, the parent PR can be merged into the base branch

## Rebasing the stack

When the base changes (e.g., audit fixes, upstream merges, or review feedback on an earlier PR), rebase the entire stack in cascade order:

1. Rebase the feature branch onto the base branch (if needed)
2. Rebase stack branch 1 onto the feature branch
3. Rebase stack branch 2 onto stack branch 1
4. Rebase stack branch 3 onto stack branch 2
5. ...and so on

Then push all branches with `--force-with-lease`:
```
git push --force-with-lease origin <feature-branch> stack/<prefix>/01-<slug> stack/<prefix>/02-<slug> stack/<prefix>/03-<slug>
```

## Important guidelines

- **Two approval gates**: Get user approval on the commit plan before writing code, and on the stacking plan before creating branches/PRs.
- **Always use `--force-with-lease`** instead of `--force` when pushing rebased branches, to avoid overwriting others' work.
- **Scope CI/format commands** to the relevant package directory, not the workspace root, to avoid side effects on other packages.
- **Keep each PR focused**: aim for under 400 lines of diff per PR when possible.
- **Cap the stack at 5-7 PRs**. If there are more commits than that, group related commits together.
- **Adapt to the repo's PR template**: if a `.github/pull_request_template.md` exists, use its section structure for the PR body (above the Stack section). If not, use a minimal summary format.
- If the user wants to add reviewers, labels, or link issues, ask before creating PRs and apply these in the `gh pr create` commands.
