---
name: create-pull-request
description: >-
  Create a GitHub pull request from the current branch into main using the gh
  CLI. Use when the user asks to open a PR, create a pull request, submit
  changes for review, or references pull requests and merging into main.
---

# Create Pull Request

Open a GitHub pull request from the current branch into `main` using the `gh`
CLI.

## Workflow

### Step 1: Ensure clean working tree

```bash
git status --porcelain
```

If there are uncommitted changes, **use the `commit-changes` skill** to commit
and push them before continuing.

### Step 2: Verify branch

```bash
git rev-parse --abbrev-ref HEAD
```

- If the branch is `main` or `master`, **stop** and ask the user to switch to a
  feature branch first.
- Push the branch if it has no upstream yet:

```bash
git push -u origin HEAD
```

### Step 3: Gather context

Run these commands to understand what the PR contains:

```bash
git log main..HEAD --oneline
git diff main...HEAD --stat
```

### Step 4: Craft the PR title

Use the same Conventional Commits style as the commit messages:

```
<type>(<optional scope>): <short summary>
```

If the branch has a single commit, reuse its message. For multiple commits,
write a title that captures the overall intent.

### Step 5: Write the PR description

Use this template **exactly** — do not add extra sections:

```markdown
# Description
One or two sentences max explaining what and why.

# Changes
- [x] Short change description
- [x] Another change

# How to test
_Scenario: <scenario name>_
1. Step one
2. Step two
```

#### Rules

| Element | Guideline |
|---------|-----------|
| **Description** | One or two sentences. State what changed and why. |
| **Changes** | Max 6 items. Each is a short, checked `- [x]` line. Consolidate related changes into a single item if needed. |
| **How to test** | One scenario with numbered steps. Keep it actionable. |
| **Additional Information** | Only add this section if truly relevant context exists (e.g. migration steps, breaking changes). Omit it otherwise. |

### Step 6: Create the pull request

```bash
gh pr create --base main --title "<title>" --body "$(cat <<'EOF'
<description>
EOF
)"
```

If `gh` is not authenticated or not installed, inform the user and stop.

### Step 7: Report back

Print the PR URL so the user can review it.

## Important Constraints

- **Never target a branch other than `main`** unless the user explicitly asks.
- **Never merge the PR** — only create it.
- **Never skip the uncommitted-changes check** (Step 1).
- **Keep wording minimal** — every sentence must earn its place.
- **Max 6 change items** — consolidate if the diff is large.
- **Stick to the template** — do not invent sections.
