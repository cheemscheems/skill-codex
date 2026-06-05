---
name: opencode
description: Use when the user asks to run OpenCode CLI (opencode run) or references OpenCode for code analysis, refactoring, or automated editing
---

# OpenCode Skill Guide

## Running a Task

1. Ask the user (via `AskUserQuestion`) which model to run (format: `provider/model`, e.g. `anthropic/claude-sonnet-4-20250514`, `openai/gpt-4.1`) AND which variant/reasoning effort to use in a **single prompt with two questions**.
2. Select the agent mode required for the task:
   - **`plan`**: for analysis, code review, exploration — requires explicit user approval before making edits or running bash commands (plan_enter/plan_exit gating). Note: this is approval-based gating, **not** filesystem-enforced read-only like Codex's `--sandbox read-only`.
   - **`build`** (full-access): for edits, refactoring, generation — can modify files and run commands without gating.
   Default to `plan` unless edits are necessary. Run `opencode agent list` to see all available agents.
3. Assemble the command with the appropriate options:
   - `-m, --model <provider/model>`
   - `--variant <reasoning-effort>`
   - `--agent <plan|build>`
   - `--dir <DIR>`
   - `--thinking` (show CoT reasoning — useful for analysis tasks)
   - `-f, --file <FILE>` (attach file to message)
   - `--format <default|json>` (omit for human-readable; use `json` for structured parsing)
   - `"your prompt here"` (as final positional argument, or use `--prompt <TEXT>`)
4. When continuing a previous session, use `opencode run --continue "your prompt here"`. To resume a specific session: `opencode run --session <session-id> "your prompt here"`. To fork a session (branch off without modifying the original): add `--fork` (requires `--continue` or `--session`). No special stdin piping needed — OpenCode handles session continuation natively.
5. **IMPORTANT**: Use `--format json` when structured output is needed for parsing. Otherwise omit — `default` is already the implicit format.
6. **IMPORTANT (stdin)**: OpenCode reads stdin in `run` mode. If invoking from a harness where stdin is not a TTY but also not closed, redirect stdin: append `</dev/null` to the command to prevent blocking.
7. Run the command, capture stdout/stderr, and summarize the outcome for the user.
8. After OpenCode completes, optionally run `opencode stats` to report token usage and cost.
9. **After OpenCode completes**, inform the user: "You can resume this OpenCode session at any time by saying 'opencode continue' or asking me to continue with additional analysis or changes."

### Quick Reference

| Use case | Agent | Key flags |
| --- | --- | --- |
| Read-only review or analysis | `plan` | `--agent plan` |
| Analysis with CoT reasoning | `plan` | `--agent plan --thinking` |
| Apply local edits | `build` | `--agent build` |
| Structured output for parsing | any | `--format json` |
| Attach file to prompt | any | `-f <file>` |
| Resume recent session | Inherited | `--continue "prompt"` |
| Resume specific session | Inherited | `--session <id> "prompt"` |
| Fork session (branch off) | Inherited | `--continue --fork "prompt"` |
| Run from another directory | Match task needs | `--dir <DIR>` plus other flags |

## Model Selection

OpenCode uses `provider/model` format. Common examples:

| Provider | Model example |
|---|---|
| Anthropic | `anthropic/claude-sonnet-4-20250514`, `anthropic/claude-opus-4-20250514` |
| OpenAI | `openai/gpt-4.1`, `openai/o3` |
| Google | `google/gemini-2.5-pro` |
| Free (OpenCode) | `opencode/deepseek-v4-flash-free`, `opencode/mimo-v2.5-free` |
| Local (Ollama) | `ollama/llama3.3` |

Run `opencode models` to list all available models from configured providers.

## Execution Timeouts

OpenCode runs synchronously in `run` mode and streams output. There is no silent-empty-output risk like Codex — you see progress as it happens.

**If running in background**, set a generous timeout since OpenCode may take time for complex tasks:

| Task complexity | Timeout |
|---|---|
| Simple query | 120s |
| Code review / analysis | 300s |
| Multi-file refactoring | 600s |
| Large-scale changes | 1200s |

## Following Up

- After every `opencode run` command, use `AskUserQuestion` to confirm next steps or decide whether to continue the session.
- When continuing, use `--continue "new prompt"` or `--session <id> "new prompt"`. Add `--fork` to branch off without modifying the original session (fork creates a new session copy). The continued session automatically uses the same model and agent from the original session.
- Restate the chosen model, variant, and agent mode when proposing follow-up actions.

## Critical Evaluation of OpenCode Output

OpenCode can use various LLM backends with their own knowledge cutoffs and limitations. Treat OpenCode as a **colleague, not an authority**.

### Guidelines

- **Trust your own knowledge** when confident. If OpenCode claims something you know is incorrect, push back directly.
- **Research disagreements** using WebSearch or documentation before accepting OpenCode's claims.
- **Remember knowledge cutoffs** — OpenCode may not know about recent releases, APIs, or changes that occurred after its training data.
- **Don't defer blindly** — OpenCode can be wrong. Evaluate its suggestions critically, especially regarding:
  - Model names and capabilities
  - Recent library versions or API changes
  - Best practices that may have evolved

### When OpenCode is Wrong

1. State your disagreement clearly to the user
2. Provide evidence (your own knowledge, web search, docs)
3. Optionally continue the OpenCode session to discuss the disagreement:
   ```bash
   opencode run --continue "I disagree with [X] because [evidence]. What's your take?"
   ```
4. Frame disagreements as discussions, not corrections — either AI could be wrong
5. Let the user decide how to proceed if there's genuine ambiguity

## Error Handling

- Stop and report failures whenever an `opencode run` command exits non-zero; request direction before retrying.
- Run `opencode --version` as a pre-flight health check if OpenCode has not been used recently in the session.
- When output includes warnings or partial results, summarize them and ask how to adjust using AskUserQuestion.
