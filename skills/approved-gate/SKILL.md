---
name: approved-gate
description: Install the approved-issue gate in a GitHub repository, so code only lands for issues a maintainer labelled `approved`. PRs without one close themselves and reopen on approval, and AI co-author trailers are blocked. Use when the user wants to adopt the tinycast contribution convention, gate PRs on approved issues, or set up a repo's contribution rules.
---

# Approved-issue gate

The rule: **a maintainer's `approved` label on an issue is the only green light to write code.** A PR from anyone but OWNER/MEMBER/COLLABORATOR that doesn't `Closes #n` an approved issue is closed automatically, and it reopens when the issue is approved. Docs-only diffs are exempt. Bots follow the same rule: open a PR only for an approved issue.

The logic lives once, as reusable workflows in [ajchemist/.github](https://github.com/ajchemist/.github) (`approved-gate.yml`, `ai-coauthors.yml`). This skill writes only the per-repo pieces. **Never copy the gate logic into the repo.**

## 1. Explore

Before writing anything, find out:

- `git remote -v`: it must be GitHub.
- `.github/workflows/`: an existing triage or gate workflow you would duplicate?
- `.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/`, `CONTRIBUTING.md`: extend these, don't replace them.
- Which paths are docs here (`*.md`, `docs/`, `website/`, …): these become `docs-pattern`.
- `gh label list`: is there already a label that means approved? If so, reuse its name as `approval-label`.
- `docs/agents/triage-labels.md`: if it exists, add the approval label there. It sits beside `ready-for-*`: `approved` means "decided"; `ready-for-*` means "who does it".

Then show the user what you found and the drafts below, and let them edit before you write.

## 2. Write

**`.github/workflows/triage.yml`**:

```yaml
name: Triage
on:
  issues:
    types: [opened, labeled]
  pull_request_target:
    types: [opened, reopened, edited, synchronize]

jobs:
  gate:
    if: github.event_name == 'issues' || github.event.action != 'synchronize'
    permissions: { issues: write, pull-requests: write }
    uses: ajchemist/.github/.github/workflows/approved-gate.yml@v1
    with:
      approval-label: approved
      docs-pattern: '\.md$|^docs/'
  ai-coauthors:
    if: github.event_name == 'pull_request_target'
    permissions: { pull-requests: write }
    uses: ajchemist/.github/.github/workflows/ai-coauthors.yml@v1
```

Leave out `with:` lines that equal the defaults.

**`.github/PULL_REQUEST_TEMPLATE.md`**: it must start with the issue link. If the repo has a template, prepend this section and keep the rest:

```markdown
## Related issue

Closes #

## What changed

## Tests & validation
```

**`.github/ISSUE_TEMPLATE/config.yml`**: `blank_issues_enabled: false`. Only when at least one issue template exists: add a minimal `request.yml` form if there is none, or nobody can open an issue.

**`CONTRIBUTING.md`**: add (or create the file with) a "Before submitting" section:

> **A linked issue marked `approved`.** Put `Closes #<number>` in the PR description. A PR that doesn't close an `approved` issue is closed automatically the moment it opens. It reopens on its own when the linked issue is approved; if the link was missing, fix it and reopen the PR. Anyone may open the PR, not only the issue's author. Docs-only changes skip the check.
>
> Commits must not credit an AI tool as a co-author (`Co-authored-by:` trailers); a check fails the PR if they do.

**Label**: `gh label create approved --color 0e8a16 --description "Maintainer approved; PRs may close this" --force`.

## 3. Done

Tell the user:

- The gate only acts on PRs from people outside the repo, and it ignores PRs opened with `GITHUB_TOKEN`, because those start no workflows.
- To change the gate itself, change `ajchemist/.github` and move the `v1` tag; every repo picks it up.
