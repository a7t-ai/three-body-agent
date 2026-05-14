# Contributing to Three-Body Agent

Thanks for considering a contribution. This project is intentionally small: shell scripts, GraphQL queries, GitHub Actions YAML, and Markdown prompt templates. No frameworks, no dependencies beyond `gh`, `jq`, and `curl`. Please keep it that way.

## Reporting bugs / requesting features

Use the issue templates under [.github/ISSUE_TEMPLATE/](./.github/ISSUE_TEMPLATE/). Bug reports need a workflow run URL or log snippet; feature requests need a use case, not just a solution.

## Submitting changes

1. Fork the repo and create a branch off `main`.
2. Keep changes focused — one logical change per PR.
3. Run `shellcheck` on any `.sh` you touched and `actionlint` on any workflow YAML you touched.
4. Test workflows by triggering them via `workflow_dispatch` on a fork before opening the PR.
5. Open the PR using the [pull request template](./.github/PULL_REQUEST_TEMPLATE.md).

## Code style

- **Shell:** POSIX-compatible where possible; reach for bashisms only when the readability win is clear.
- **YAML:** 2-space indent, lowercase keys, double quotes for strings.
- **Markdown prompts:** lead with the goal in one sentence, then constraints, then the output format. No unnecessary preamble.

## Things we generally won't accept

- Dependencies on agent frameworks (LangGraph, CrewAI, AutoGen, ...). The pipeline is shell-only by design.
- Hard-coded org / repo / project values. Use `${{ env.* }}` references so downstream users can configure their own.
- Workflows that assume Claude Code CLI is the only model. The pipeline should keep room for OpenAI-compatible endpoints (MLX, Ollama, vLLM, Groq, ...).

## Code of Conduct

This project follows the [Contributor Covenant](./CODE_OF_CONDUCT.md). Be kind, be precise, be useful.
