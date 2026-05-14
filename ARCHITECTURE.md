# Architecture

The decisions behind Three-Body Agent, and how it compares to Anthropic's Managed Agents.

## Architecture decisions

**Why shell scripts instead of a framework?** The orchestration is simple enough that shell + `gh` + `jq` covers it. No dependency management, no build step, no abstraction layers. The intelligence comes from the model, not the framework.

**Why sequential merging?** Merging PR A can create conflicts in PR B. Sequential merging with re-verification catches this. The fixer handles the rebase on its next cycle.

**Why milestone filtering?** Without it, the implementer picks from the entire backlog. Milestones scope work to the current sprint, matching how teams actually plan.

**Why `AGENT_PAT` instead of `GITHUB_TOKEN`?** The default `GITHUB_TOKEN` cannot access organization-level GitHub Projects V2. A PAT with `project` scope is required.

**Why not [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action)?** Anthropic's official GitHub Action is excellent for interactive use cases — PR review, `@claude` mentions, issue triage, and scoped automation. It's not the right fit for long-running autonomous agents:

- **Coarse permission model.** The action controls tool access via explicit allowlists (`--allowedTools`). Autonomous agents that read files, run tests, install dependencies, and push code would require enumerating every allowed Bash command pattern — fragile and hard to maintain.
- **Designed for shorter interactions.** It's optimized for PR review and targeted fixes, not 90-minute sessions with 500 max turns doing full-issue implementation.
- **Orchestration lives outside Claude.** The real complexity — board scanning, priority ranking, milestone filtering, dependency detection, GraphQL mutations, Telegram notifications — is shell logic that runs before and after the Claude step. The action doesn't help with any of that.
- **Direct CLI gives full control.** Installing Claude Code via `npm install -g @anthropic-ai/claude-code` and calling it directly is simpler, more transparent, and doesn't add an abstraction layer.

The action is the right choice for interactive `@claude` workflows. For autonomous agents, the CLI is the right tool.

## `--permission-mode`

Claude Code CLI offers several permission modes for non-interactive use:

| Mode                | Behavior                                                                                                                                                                                  | Use case                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `auto`              | Runs a safety classifier that reviews every action, blocking dangerous operations (external code execution, mass deletion, force push, etc.) while allowing normal development work       | **Recommended for CI.** Used by this template. |
| `dontAsk`           | Only runs pre-approved tools from `--allowedTools` / `permissions.allow` rules. Auto-denies everything else                                                                               | Locked-down CI with strict tool control        |
| `bypassPermissions` | Skips all permission prompts and safety checks                                                                                                                                            | Isolated containers/VMs with no internet only  |

This template uses `auto`. It gives the agent full autonomy for normal development tasks (file edits, running tests, git operations) while the classifier blocks genuinely dangerous actions. If the classifier blocks an action repeatedly, the session aborts — a safe failure mode for headless runs.

If you run on fully isolated, disposable runners (Docker containers, VMs), you can switch to `bypassPermissions` for zero friction. For shared or persistent runners, `auto` is the right default.

> **Warning:** Using `bypassPermissions` on company projects can be grounds for termination — it disables all safety checks, which may violate your organization's security policies. Always check with your employer before using it.

## Relationship to Claude Managed Agents

Anthropic launched [Claude Managed Agents](https://www.anthropic.com/products/managed-agents) on the same day this project was released. The overlap is real — and intentional validation that autonomous development pipelines are the next frontier.

**What Managed Agents provides.** Cloud-hosted Claude sessions on a schedule, sandboxed execution, session persistence, and GitHub access via MCP tools. It solves the infrastructure problem: _how do I run Claude autonomously?_

**What Three-Body Agent provides.** The orchestration logic that turns autonomous Claude sessions into a functioning development team. Priority-based issue selection, dependency detection, sequential merge strategy, conflict-aware fixing, board state management, sprint automation, and multi-agent coordination with deduplication and concurrency controls.

Managed Agents is the engine. Three-Body Agent is the self-driving car.

You can run this pipeline on GitHub Actions (as shipped), or adapt the workflow logic to run on Managed Agents infrastructure. The shell scripts, GraphQL queries, and prompt templates are the actual value — they work regardless of where Claude runs.
