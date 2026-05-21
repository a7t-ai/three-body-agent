<div align="center">

<img src="assets/hero.png" alt="Three-Body Agent" width="600" />

# Three-Body Agent

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**An autonomous development pipeline powered by GitHub Actions and Claude Code CLI. Five workflows pick issues from a project board, implement them, fix their own CI failures, merge green PRs, and keep the board updated — all without human intervention.**

</div>

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

Three systems in constant gravitational pull, each with its own orbit, producing stable results. Shell scripts and GraphQL — no framework, no SDK, no dependencies beyond `gh`, `jq`, and `curl`. The intelligence comes from the model.

> **The Playbook.** The full architecture behind this repo is written down in [*Three-Body Agent: the Playbook*](https://a7t.ai/booklets/three-body-agent-playbook/): every design decision, the verbatim role prompts, the failure catalogue, and the cost dashboards. This repo is the runnable source; the book is the reasoning.

## 60-second quickstart

1. Copy `.github/workflows/` and `.github/prompts/` into your repository.
2. Add three secrets (`Settings → Secrets and variables → Actions`):
   - `ANTHROPIC_API_KEY` — Claude Code CLI is installed automatically by the workflows.
   - `AGENT_PAT` — a PAT with `repo`, `project`, `workflow` scopes (org-level Projects V2 needs this; `GITHUB_TOKEN` can't reach it).
   - `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` — optional. Swap for Slack / Discord / email by editing the `curl` call in `telegram.yml`.
3. Create a Projects V2 board with the columns `Todo`, `In Progress`, `Ready For QA`, `Done`. Add priority labels `p0` (critical) through `p5` (backlog). Optional: milestones in `"YY CW WW"` format for sprint scoping.
4. Search every workflow file for `TODO` comments and fill in your org, repo, project number, runner labels, dependency-install command, and merge method.
5. Open an issue with clear acceptance criteria, drop it in `Todo` with a priority label, and the next hourly Implementer run picks it up.

The deeper rationale — and the comparison to Anthropic's Managed Agents — lives in [ARCHITECTURE.md](ARCHITECTURE.md).

## The five workflows

| Workflow                                       | Schedule                       | Job                                                                                                       |
| ---------------------------------------------- | ------------------------------ | --------------------------------------------------------------------------------------------------------- |
| **Implementer** (`autoagent-implementer.yml`)  | Hourly                         | Picks the highest-priority `Todo` issue in the current milestone, branches, implements, opens PR.         |
| **Fixer** (`autoagent-fixer.yml`)              | Every 30 min + on CI failure   | Handles CI failures, review comments, and merge conflicts on `autoagent/*` branches.                      |
| **Merger** (`autoagent-merger.yml`)            | Every 2 hours                  | Sequentially merges fully-green autoagent PRs; uses Claude to weigh review comments before merging.       |
| **Board Sync** (`autoagent-board-sync.yml`)    | PR events                      | Moves issues between `Todo` ↔ `In Progress` ↔ `Ready for QA` ↔ `Done`.                                    |
| **Week Rollover** (`auto-week-rollover.yml`)   | Mondays 06:00 UTC              | Rolls open issues into next week's milestone.                                                             |

`telegram.yml` is a reusable notification workflow each of the others calls — pluggable, swap the `curl` body for any messaging service.

## Writing good issues

The number-one predictor of autonomous success is issue quality:

- **Clear acceptance criteria** — what does "done" look like?
- **Example inputs/outputs** — concrete, not abstract.
- **References** — `see src/services/auth.ts for the pattern`.
- **Dependencies** — `Depends on: #123` to chain branches.

## Branch naming

`autoagent/<issue-number>-<slug>` (e.g. `autoagent/123-add-user-auth`). The issue-number prefix is required — Board Sync uses it to map branches back to issues.

## Demo

<video src="https://github.com/user-attachments/assets/2aa1bf2d-b856-4339-b230-372007655d21" controls></video>

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and the [Code of Conduct](CODE_OF_CONDUCT.md).

## Apps Built With Three-Body Agent

These shipping apps use this pipeline to ship features autonomously:

| [FareHawk](https://farehawk.app) | [Kinderuntersuchungsheft](https://kinderuntersuchungsheft.com) | [Einbürgerung Pro](https://einbuergerung.pro) | [Reellette](https://reellette.app) |
| :---: | :---: | :---: | :---: |
| <a href="https://farehawk.app"><img src="https://leocardz.com/assets/images/apps/farehawk.png" width="96" alt="FareHawk" /></a> | <a href="https://kinderuntersuchungsheft.com"><img src="https://leocardz.com/assets/images/apps/u-heft.png" width="96" alt="Kinderuntersuchungsheft" /></a> | <a href="https://einbuergerung.pro"><img src="https://leocardz.com/assets/images/apps/einbuergerung-pro.png" width="96" alt="Einbürgerung Pro" /></a> | <a href="https://reellette.app"><img src="https://leocardz.com/assets/images/apps/reellette.png" width="96" alt="Reellette" /></a> |

## License

[MIT](LICENSE)
