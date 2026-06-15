<!-- agent-workflow:start -->
# Repository Agent Instructions

Use this file as the common bootstrap for Codex, GitHub Copilot, Cursor agents, and GitNexus. Keep detailed implementation rules in the Cursor rule files and GitNexus index; do not duplicate those source files here.

## Before Starting Any Task

1. Identify the repository you are actually changing.
   - In this repository (`/Users/sna/Desktop/projects/integration`), use GitNexus repo `ai-managing`.
   - In the ADEO frontend repository (`/Users/sna/Desktop/adeo-projects/frontend`), use GitNexus repo `frontend`.
   - If an instruction block was copied from another repository, update the GitNexus repo name and paths before using it.
2. Read the base Cursor rules when they exist:
   - Prefer `.cursor/rules/basic.mdc` and `.cursor/rules/technical.mdc`.
   - If those files are not present, use `cursor/rules/basic.mdc` and `cursor/rules/technical.mdc`.
3. If the user provides labels, read the matching rule file before planning or implementing:
   - `[COMPONENT]` -> `component.mdc`
   - `[API]` -> `api.mdc`
   - `[FORM]` -> `form.mdc`
   - `[LANG]` -> `translations.mdc`
   - `[E2E]` -> `e2e.mdc`
   - `[E2E_ENERGO]` -> `e2e-energo.mdc`
   - `[E2E_ADMIN]` -> `e2e-admin.mdc`
   - `[ENERGO]` -> `energo.mdc`
   - `[GRAPH]` -> `graph.mdc`
   - `[WCAG]` -> `wcag.mdc`
   - `[CODE]` -> `code.mdc`
   - `[BUMP]` -> `bump-packages.mdc`
   - `[FIGMA]` -> `figma.mdc`
   - `[JIRA]` -> `jira.mdc`
4. If selected rules contain `ASK` or `VERIFY`, follow those instructions before implementation.
5. For code changes, prepare the task list requested by the repository rules and wait for user approval when those rules require it.
6. Follow the target repository's existing Nx, Nuxt, Vue, TypeScript, FSD, Pinia, Vuetify/Mozaic, Vitest, and Playwright conventions when working in the frontend codebase.

## Agent Cooperation Workflow

- Start with the repository rules, then use GitNexus for code intelligence.
- For unfamiliar code, run a GitNexus query before broad text search so work starts from execution flows instead of isolated files.
- Before editing a function, class, method, API route handler, or shared contract, run GitNexus impact analysis for the correct repo and report the blast radius to the user.
- If impact analysis is `HIGH` or `CRITICAL`, warn the user and wait for confirmation before editing.
- Prefer GitNexus rename/refactor tooling for symbol renames; do not use plain find-and-replace for code symbols.
- Before committing, run GitNexus change detection and confirm the affected symbols and execution flows match the intended scope.
- When handing work between Codex, GitHub Copilot, or Cursor, include: rules read, labels used, GitNexus repo queried, impact result, files changed, tests run, and remaining risks.

## AIConnector MCP Workflow Sessions

- Use `MCP.md` as the detailed workflow-session contract for `flow_runner_mcp`.
- Every LLM call to AIConnector MCP, workflow-session tools, workflow runs, chain runs, remote runs, and graph processing must include a `chat_messages` payload field whenever the tool accepts a body/input/options object.
- When an AIConnector workflow session exists, also treat `workflow_session_event_append` as the special MCP logging tool for standalone transcript logging. Use it to append `llm_chat_messages` / `llm_transcript` events before or after MCP actions when the main MCP tool cannot carry `chat_messages`, or when the transcript must be logged separately for audit.
- `chat_messages` is the canonical transcript field. It should contain the conversation and execution context available to the LLM: user requests, assistant messages, explicit reasoning notes that may be logged, summaries, decisions, MCP/tool calls, workflow/chain actions, results, errors, and handoff notes.
- Keep `chat_messages` current before each MCP call. When a workflow, chain, graph node, or session process continues separately from the original LLM turn, pass the latest transcript forward so downstream processing can log what led to that action.
- Do not replace or shrink an existing `chat_messages` array unless the user explicitly asks for redaction. Append new LLM-visible steps in order and keep each entry structured with at least `role` and `content`.
- Use aliases such as `chatMessages` or `messages` only for compatibility with older callers. New agent instructions and examples should use `chat_messages`.
- Preferred standalone logging event:

```json
{
  "sessionId": "wfs_...",
  "type": "llm_chat_messages",
  "title": "LLM transcript update",
  "status": "ok",
  "body": {
    "chat_messages": [
      {
        "role": "assistant",
        "content": "Summarized the next MCP action and prepared the workflow payload.",
        "type": "summary"
      }
    ]
  },
  "metadata": {
    "tool": "codex",
    "purpose": "mcp_transcript_log"
  }
}
```

- When a session needs human data or approval, do not finish the agent turn with only a note. Create a visible Ask log entry with `workflow_session_ask_user` or `workflow_session_request_user_input`.
- Ask payloads should include `question`, `requiredFields`, and `optionalFields` when known. The browser user answers from the session timeline by clicking **Answer question**, filling the prepared body, and appending it as `user_answer`.
- When any workflow, chain, graph node, or MCP response returns `status: "paused"`, `paused: true`, `session.status: "waiting-for-user"`, or a `user_question`/`askLog`, stop advancing downstream workflows. Treat it as a coordinated pause, not a failed run.
- Do not start the next workflow in a chain while the active session is `waiting-for-user` or `waiting-for-llm`. Resume only after a browser user answer or an LLM-generated fill is recorded as `user_answer` or `task_enqueued`.
- After creating an Ask log, keep a stable wait open with `workflow_session_wait_for_user_answer` and `processNext: true`, or give the operator the returned `session.terminalWait.command` curl and tell them to keep it running.
- Use the latest known `session.eventCount` as `cursor` so old answers are not replayed.
- Preferred MCP wait payload:

```json
{
  "sessionId": "wfs_...",
  "cursor": 52,
  "timeoutSeconds": 1800,
  "processNext": true,
  "autopilot": true,
  "parallel": false,
  "chat_messages": [
    {
      "role": "user",
      "content": "Original user request"
    },
    {
      "role": "assistant",
      "content": "Summary of the LLM steps already taken before waiting for the browser user."
    }
  ]
}
```

- Equivalent curl pattern for a terminal wait:

```bash
curl -sS -N -X POST 'http://127.0.0.1:8790/api/workflow-sessions/wfs_.../wait-for-user-answer' \
  -H 'content-type: application/json' \
  --data '{"cursor":52,"timeoutMs":1800000,"processNext":true,"autopilot":true,"parallel":false,"chat_messages":[{"role":"assistant","content":"Waiting for the browser user after asking for missing data."}]}'
```

- If the wait times out and the user did not ask to stop, repeat the same wait with the returned cursor. If the processed result returns `waiting-for-user` again, start another Ask/wait loop.
- If an LLM model can answer the gap itself, append that answer to the same session as `user_answer` with the requested fields and then call `workflow_session_process_next` or the wait endpoint with `processNext: true`. Keep the event body explicit so the browser timeline shows what was filled.

## Local Rule Paths

This integration repository stores reusable Cursor content under `cursor/`. Some target repositories store the same content under `.cursor/`. Agents should prefer the target repository's own files and only fall back to this repository's templates when explicitly preparing or updating shared agent instructions.

## Update docker images after updating repository code, that is needed for tracking proper code
- needs to be updated proper docker image after changes made
- needs to be suggested for a user how to use the new funcionality

<!-- agent-workflow:end -->

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **ai-managing** (20379 symbols, 35034 relationships, 300 execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> If any GitNexus tool warns the index is stale, run `npx gitnexus analyze` in terminal first.

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `gitnexus_impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `gitnexus_detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `gitnexus_query({query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `gitnexus_context({name: "symbolName"})`.

## Never Do

- NEVER edit a function, class, or method without first running `gitnexus_impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `gitnexus_rename` which understands the call graph.
- NEVER commit changes without running `gitnexus_detect_changes()` to check affected scope.

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/ai-managing/context` | Codebase overview, check index freshness |
| `gitnexus://repo/ai-managing/clusters` | All functional areas |
| `gitnexus://repo/ai-managing/processes` | All execution flows |
| `gitnexus://repo/ai-managing/process/{name}` | Step-by-step execution trace |

## CLI

| Task | Read this skill file |
|------|---------------------|
| Understand architecture / "How does X work?" | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| Blast radius / "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| Trace bugs / "Why is X failing?" | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / split / refactor | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, resources, schema reference | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index, status, clean, wiki CLI commands | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |

<!-- gitnexus:end -->

@RTK.md
