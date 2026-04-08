# create-stacked-diff-prs

A [Claude Code skill](https://code.claude.com/docs/en/skills.md) for splitting feature branches into stacked diff PRs.

## What it does

Breaks a large feature branch into a series of small, dependent pull requests that merge into the feature branch (not directly into main). A parent PR from the feature branch to the base branch ties everything together.

```
main <-- feature-branch (parent PR, closes the issue)
            ^
            |-- stack/feature/01-models   (PR targeting feature-branch)
            |      ^
            |      |-- stack/feature/02-api   (PR targeting 01-models)
            |             ^
            |             |-- stack/feature/03-tests   (PR targeting 02-api)
```

## Install

```bash
npx skills add widal001/create-stacked-diff-prs
```

Or manually copy `SKILL.md` to `.claude/skills/create-stacked-diff-prs/SKILL.md` in your project (or `~/.claude/skills/` for global use).

## Usage

In Claude Code, type:

```
/create-stacked-diff-prs
```

Or describe what you want and Claude will invoke the skill automatically:

> "Split this branch into stacked PRs"
> "Break up this branch for easier review"

## How it works

1. **Gather context**: Analyzes commits, files changed, and the repo's PR template
2. **Plan the commits** (approval gate): Proposes how to organize work into clean, squash-ready commits
3. **Plan the stack** (approval gate): Proposes how to split commits into 2-5 incremental PRs
4. **Create branches**: Builds stack branches incrementally via cherry-pick
5. **Create PRs**: Parent PR (feature branch to main) + stack PRs with cross-linked descriptions
6. **Backfill**: Updates all PR descriptions with final PR numbers

## Key design decisions

- **Stack PRs merge into the feature branch**, not main. This lets you test the full feature before merging to main.
- **Squash-and-merge strategy**: Each stack PR is squash-merged in order. Downstream PRs are rebased after each merge.
- **Two approval gates**: The skill pauses for user confirmation before writing code and again before creating PRs.
- **Adapts to repo conventions**: Uses the repo's PR template if one exists.

## License

MIT
