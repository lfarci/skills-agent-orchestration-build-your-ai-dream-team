# Agent team

To build Mona's Project Pulse dashboard, I use a four-agent team defined under `.github/agents/`, coordinated through GitHub Copilot CLI running in a Codespace.

## Orchestrator

- **Model:** Claude Opus 4.7 (copilot)
- **Definition:** `.github/agents/orchestrator.agent.md`
- **Responsibility:** Coordinates the Planner, Coder, and Designer agents. Breaks requests into phases, assigns non-overlapping file scopes, decides what can run in parallel vs. sequentially, and reports the integrated outcome. Does not implement work itself and never stages, commits, or pushes changes.

## Planner

- **Model:** Claude Opus 4.7 (copilot)
- **Definition:** `.github/agents/planner.agent.md`
- **Responsibility:** Researches the codebase, documentation, dependencies, and edge cases, then produces an implementation plan (steps, file assignments, dependencies, parallelizable work, validation expectations, and open questions) for the Orchestrator to turn into phases. Does not write code.

## Coder

- **Model:** GPT-5.5 (copilot)
- **Definition:** `.github/agents/coder.agent.md`
- **Responsibility:** Implements code-oriented tasks within the file scope assigned by the Orchestrator, including Project Pulse support files such as `.vscode/launch.json`. Follows existing repository patterns, keeps behavior deterministic and testable, and validates changes before reporting completion.

## Designer

- **Model:** Gemini 3.1 Pro (copilot)
- **Definition:** `.github/agents/designer.agent.md`
- **Responsibility:** Handles UI/UX, accessibility, information architecture, interaction flow, and visual design for Project Pulse, producing a polished dashboard (project cards, status badges, responsive layout, deterministic CSS hooks like `.dashboard` and `.project-card`) within the scope assigned by the Orchestrator.

## Orchestration note

I use GitHub Copilot CLI in a Codespace to drive this team: the Orchestrator agent delegates to Planner, Coder, and Designer, while I (the learner) retain control of all git operations (staging, committing, and pushing).
