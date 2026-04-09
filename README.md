# Three-Body Agent

An autonomous development pipeline powered by GitHub Actions and Claude Code CLI. Five workflows that pick issues from a project board, implement them, fix their own CI failures, merge green PRs, and keep the board updated - all without human intervention.

```
┌─────────────────────────────────────────────────────┐
│                  GitHub Actions                     │
│              (Autonomous Brain)                     │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ Implementer │  │    Fixer    │  │   Merger    │  │
│  │  (hourly)   │  │ (every 30m) │  │ (every 2h)  │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐                   │
│  │ Board Sync  │  │  Rollover   │                   │
│  │ (PR events) │  │  (weekly)   │                   │
│  └─────────────┘  └─────────────┘                   │
│                                                     │
│              Claude Code CLI                        │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────┴─────────────────────────────┐
│              GitHub Projects V2                     │
│            (State Management)                       │
│                                                     │
│   Board: Todo → In Progress → Ready for QA → Done   │
│   Milestones: "26 CW 14", "26 CW 15", ...           │
│   Labels: p0 (critical) through p5 (backlog)        │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────┴─────────────────────────────┐
│              Telegram Notifications                 │
│         (Visibility at every stage)                 │
└─────────────────────────────────────────────────────┘
```

## How It Works

Three systems in constant gravitational pull, each with its own orbit, producing stable results:

1. **GitHub Actions** runs the autonomous workflows on schedule
2. **GitHub Projects V2** is the shared state - board columns, milestones, and labels
3. **Telegram** provides visibility at every stage of the pipeline

The entire system is shell scripts and GraphQL queries. No framework, no SDK, no dependencies beyond `gh`, `jq`, and `curl`. Everything is auditable in the GitHub Actions logs and version-controlled alongside the code it operates on.

## Workflows

### [AUTOAGENT] Implementer

**Schedule**: Every hour | **File**: `autoagent-implementer.yml`

Scans the project board for TODO issues in the current week's milestone, picks the highest-priority one, and hands it to Claude Code CLI for autonomous implementation.

**Pipeline**: Scan board → Pick issue (priority + milestone) → Move to In Progress → Create branch → Implement → Test → Open PR → Notify

Key features:

- **Priority ranking**: Labels `p0` through `p5`. Sorts by priority first, then by issue body length (better-specified issues = higher success rate)
- **Milestone filtering**: Only picks issues assigned to the current calendar week
- **Dependency detection**: If an issue body contains "Depends on: #123", the implementer bases the branch on #123's open PR instead of main
- **Safety checks**: Skips if anything is already In Progress or another implementer is running

### [AUTOAGENT] Fixer

**Schedule**: Every 30 min at :15 and :45 | **Trigger**: CI failure on autoagent branches | **File**: `autoagent-fixer.yml`

When CI fails or a reviewer requests changes on an autoagent PR, the fixer gathers all failure context and feeds it to Claude for autonomous fixing.

**Handles three failure types**:

1. **CI failures**: Reads failed run logs, finds root cause, fixes the actual bug
2. **Code review comments**: Addresses each comment, fixes critical/medium issues
3. **Merge conflicts**: Rebases on base branch, resolves conflicts, pushes with `--force-with-lease`

Key features:

- **Dedup**: Queries active fixer workflow runs to skip PRs already being fixed
- **Auto-trigger**: Fires on `check_suite` failures for autoagent branches
- **Safety**: Only touches `autoagent/*` branches, never human PRs
- **Re-requests reviewers** after pushing fixes

### [AUTOAGENT] Merger

**Schedule**: Every 2 hours | **File**: `autoagent-merger.yml`

Scans for autoagent PRs that are fully green (all checks pass, no changes requested, no conflicts, no unresolved review issues) and merges them sequentially.

Key features:

- **Sequential merging**: Processes PRs one at a time, re-verifying status before each merge
- **Conflict awareness**: After merging one PR, re-checks the next - if it now has conflicts, skips it (the fixer will handle rebase on its next run)
- **Claude merge analysis**: Before merging, Claude analyzes review comments, verdicts, and recent commits to determine if CRITICAL/MEDIUM issues were addressed - smarter than keyword matching. These issues can come from GitHub Copilot reviews, Claude Code reviews, or any human reviewer
- **Fast pre-filter**: Keyword scan in the prepare phase catches obvious blockers before invoking Claude
- **Fail-open**: If Claude is unavailable, defaults to merge (mechanical checks already passed)
- **Concurrency group**: Prevents overlapping merger runs

### [AUTOAGENT] Board Sync

**Trigger**: PR events on autoagent branches | **File**: `autoagent-board-sync.yml`

Keeps the project board in sync with PR state:

- **PR opened** → Issue moves to "Ready for QA"
- **PR merged** → Issue moves to "Done"
- **PR closed without merge** → Issue moves back to "Todo"

### [AUTO] Week Rollover

**Schedule**: Every Monday at 06:00 UTC | **File**: `auto-week-rollover.yml`

Automates sprint management with two modes:

- **Milestones mode** (default): Creates next week's milestone, moves open issues forward, closes the old milestone
- **Iterations mode**: Uses GitHub Projects V2 iteration fields instead of milestones

Milestone format: `"YY CW WW"` (e.g., `"26 CW 14"`)

### Telegram Notifications

**Reusable workflow** | **File**: `telegram.yml`

Every workflow sends notifications at start and completion. Silence is the worst signal for autonomous agents - you can't tell the difference between "nothing happened" and "everything is broken."

## Setup

### Prerequisites

- A GitHub repository with [GitHub Projects V2](https://docs.github.com/en/issues/planning-and-tracking-with-projects)
- A GitHub runner (self-hosted or GitHub-hosted) with `gh`, `jq`, and `curl`
- An [Anthropic API key](https://console.anthropic.com/) - Claude Code CLI is installed automatically by the workflows

### 1. Copy the workflows

Copy the `.github/workflows/` and `.github/prompts/` directories to your repository. The prompts are standalone markdown files with `${VAR}` placeholders -- edit them to match your project without touching workflow YAML.

### 2. Configure secrets

Add these secrets to your repository (Settings → Secrets and variables → Actions):

| Secret               | Description                                                                                                                                                          |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ANTHROPIC_API_KEY`  | Anthropic API key for Claude Code CLI. Add as a repository secret - the workflows inject it automatically.                                                           |
| `AGENT_PAT`          | GitHub Personal Access Token with `repo`, `project`, and `workflow` scopes. Required for org-level project board access (`GITHUB_TOKEN` cannot access org ProjectV2). |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot API token (from [@BotFather](https://t.me/BotFather)). Optional - see note below.                                                                       |
| `TELEGRAM_CHAT_ID`   | Telegram chat ID for notifications. Optional - see note below.                                                                                                       |

> **Notifications are pluggable.** The template uses Telegram, but any messaging service works - Slack, Discord, email, etc. Just swap the `curl` call in `telegram.yml` with your preferred webhook or API. The reusable workflow pattern stays the same.

### 3. Configure the workflows

Search for `TODO` comments across all workflow files. Here's what you need to change:

#### Organization and repository

In `autoagent-implementer.yml`, `autoagent-board-sync.yml`, and `auto-week-rollover.yml`:

```yaml
env:
  PROJECT_ORG: your-org # ← Your GitHub org or username
  REPO: your-org/your-repo # ← Your org/repo
  PROJECT_NUMBER: 1 # ← Your GitHub Projects V2 number
```

#### Project board columns

In `autoagent-board-sync.yml`, adjust the status names to match your board:

```bash
# These must match your project board column names exactly:
# "Todo", "In Progress", "Ready For QA", "Done"
```

#### Milestone format

In `autoagent-implementer.yml` and `auto-week-rollover.yml`, the default milestone format is `"YY CW WW"` (e.g., `"26 CW 14"`). Adjust if your convention differs, or remove milestone filtering if you don't use milestones.

#### Dependency installation

In `autoagent-implementer.yml` and `autoagent-fixer.yml`, uncomment and adjust the dependency installation step:

```yaml
# - name: Install dependencies
#   run: npm install
```

#### Runner environment

All workflows default to `ubuntu-latest`. If using self-hosted runners, change:

```yaml
runs-on: ubuntu-latest # ← Change to [self-hosted, macOS] or your runner labels
```

#### Merge method

In `autoagent-merger.yml`, the default is merge commits:

```bash
gh pr merge "$PR_NUM" --merge  # Change to --squash or --rebase
```

### 4. Set up the project board

Create a GitHub Projects V2 board with these columns:

- **Todo** - Issues ready for implementation
- **In Progress** - Currently being worked on
- **Ready For QA** - PR opened, awaiting review
- **Done** - Merged

### 5. Set up labels

Create priority labels on your repository:

- `p0` - Critical / blocker
- `p1` - High priority
- `p2` - Normal priority
- `p3` - Medium priority
- `p4` - Low priority
- `p5` - Backlog

The implementer sorts by these labels when picking the next issue.

### 6. Set up milestones (optional)

If using milestone-based filtering, create milestones with the format `"YY CW WW"` (e.g., `"26 CW 14"`). The week rollover workflow automates this - run it once manually to bootstrap.

## Writing Good Issues

The number one predictor of autonomous implementation success is the quality of the issue description. Include:

- **Clear acceptance criteria** - what does "done" look like?
- **Example inputs/outputs** - concrete examples, not abstract descriptions
- **References to existing code** - "see `src/services/auth.ts` for the pattern"
- **Dependencies** - "Depends on: #123" if this issue builds on another

## Branch Naming

The pipeline uses the `autoagent/` prefix for all automated branches:

```
autoagent/123-add-user-authentication
autoagent/456-fix-date-parsing
```

The issue number prefix is required - Board Sync uses it to map branches back to issues.

## Architecture Decisions

**Why shell scripts instead of a framework?** The orchestration is simple enough that shell + `gh` + `jq` covers it. No dependency management, no build step, no abstraction layers. The intelligence comes from the model, not the framework.

**Why sequential merging?** Merging PR A can create conflicts in PR B. Sequential merging with re-verification catches this. The fixer handles the rebase on its next cycle.

**Why milestone filtering?** Without it, the implementer picks from the entire backlog. Milestones scope work to the current sprint, matching how teams actually plan.

**Why `AGENT_PAT` instead of `GITHUB_TOKEN`?** The default `GITHUB_TOKEN` cannot access organization-level GitHub Projects V2. A PAT with `project` scope is required.

**Why not [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action)?** Anthropic's official GitHub Action is excellent for interactive use cases - PR review, `@claude` mentions, issue triage, and scoped automation. However, it's not the right fit for long-running autonomous agents:

- **Coarse permission model.** The action controls tool access via explicit allowlists (`--allowedTools`). Autonomous agents that need to read files, run tests, install dependencies, and push code would require enumerating every allowed Bash command pattern - fragile and hard to maintain.
- **Designed for shorter interactions.** The action is optimized for PR review and targeted fixes, not 90-minute sessions with 500 max turns doing full-issue implementation.
- **Orchestration lives outside Claude.** The pipeline's real complexity - board scanning, priority ranking, milestone filtering, dependency detection, GraphQL mutations, Telegram notifications - is shell logic that runs before and after the Claude step. The action doesn't help with any of that.
- **Direct CLI gives full control.** Installing Claude Code via `npm install -g @anthropic-ai/claude-code` and calling it directly is simpler, more transparent, and doesn't add an abstraction layer between your workflow and the tool.

The action is the right choice for interactive `@claude` workflows in PRs and issues. For autonomous agents, the CLI is the right tool.

**A note on `--permission-mode`.** Claude Code CLI offers several permission modes for non-interactive use:

| Mode                | Behavior                                                                                                                                                                            | Use case                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `auto`              | Runs a safety classifier that reviews every action, blocking dangerous operations (external code execution, mass deletion, force push, etc.) while allowing normal development work | **Recommended for CI.** Used by this template. |
| `dontAsk`           | Only runs pre-approved tools from `--allowedTools` / `permissions.allow` rules. Auto-denies everything else                                                                         | Locked-down CI with strict tool control        |
| `bypassPermissions` | Skips all permission prompts and safety checks                                                                                                                                      | Isolated containers/VMs with no internet only  |

This template uses `auto` mode. It gives the agent full autonomy for normal development tasks (file edits, running tests, git operations) while the classifier blocks genuinely dangerous actions. If the classifier blocks an action repeatedly, the session aborts - a safe failure mode for headless runs.

If you run on fully isolated, disposable runners (Docker containers, VMs), you can switch to `bypassPermissions` for zero friction. But for shared or persistent runners, `auto` is the right default.

> **Warning:** Using `bypassPermissions` on company projects can be grounds for termination - it disables all safety checks, which may violate your organization's security policies. Always check with your employer before using it.


<video src="https://github.com/user-attachments/assets/2aa1bf2d-b856-4339-b230-372007655d21" controls></video>

## License

MIT
