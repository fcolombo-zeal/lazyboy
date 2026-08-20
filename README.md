# LazyBoy

A tool for lazy developers to help fix bugs on a large codebase.

LazyBoy turns an Azure DevOps bug into a reviewed, branch-ready fix — harvesting context from the work item and Application Insights, working out which repos are implicated, investigating with a read-only agent, and then producing a fix on a `bug/<id>-<slug>` branch. You approve every irreversible step.

It runs locally on your machine and uses Claude Code (via the Claude Agent SDK) as its reasoning and code-editing engine.

## Design documentation

The design lives in [`docs/`](docs/). Start with the master design document:

**→ [docs/LazyBoy-Design.md](docs/LazyBoy-Design.md)**

| | |
|---|---|
| [Phase docs](docs/phases/) | Ten independently shippable phases, from the app skeleton through to PR creation |
| [Reference docs](docs/reference/) | Data model, external API cookbook, and the agent contracts |
| [Docs index](docs/README.md) | Suggested reading orders |

Status: **design v1.1** — architecture settled, all thirteen environment constraints answered. Implementation not yet started.
