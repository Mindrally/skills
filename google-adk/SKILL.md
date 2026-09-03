---
name: google-adk
description: "Best practices for building AI agents with Google's Agent Development Kit (ADK) in Python, covering agent design, tools, sessions, memory, artifacts, evaluation, and deployment. Use when building LLM agents or multi-agent systems with ADK, defining ADK tools, wiring up sessions/state/memory, working with ADK artifacts, writing agent evals, or deploying ADK agents to Vertex AI Agent Engine or Cloud Run."
---

# Google Agent Development Kit (ADK)

This skill covers building production-grade AI agents with Google's Agent Development Kit (ADK) for Python, including agent composition, tool design, session/state/memory management, artifacts, evaluation, and deployment.

## Workflow for Building an ADK Agent

1. **Define the agent** — Create an `LlmAgent` (or `Agent`) with a clear `name`, `model`, `instruction`, and `description`. Keep the instruction focused on one job.
2. **Author tools** — Write plain Python functions with type hints and docstrings, or wrap existing APIs with `FunctionTool`. Validate all inputs before side effects.
3. **Compose multi-agent systems** — For complex workflows, split responsibility across sub-agents and use `SequentialAgent`, `ParallelAgent`, or `LoopAgent` for deterministic orchestration, or delegate via `sub_agents` for LLM-driven routing.
4. **Wire up session and state** — Choose a `SessionService` (in-memory for dev, `DatabaseSessionService` or Vertex AI-managed for production) and use `session.state` for conversation-scoped data.
5. **Add memory (optional)** — Configure a `MemoryService` for cross-session recall when the agent needs to remember facts between separate conversations.
6. **Handle artifacts (optional)** — Configure an `ArtifactService` when the agent generates or receives files, images, or other binary outputs.
7. **Run locally** — Use `adk web`, `adk run`, or the `Runner` API to exercise the agent against a `Session`.
8. **Evaluate** — Write `.evalset.json` test cases and run `adk eval` to check tool-call trajectories and response quality against regressions.
9. **Deploy** — Package the agent for Vertex AI Agent Engine, Cloud Run, or GKE, and separate dev/staging/prod configuration.

## Agent Design

- Keep each agent focused on a single clear goal, persona, and tool set — avoid one agent that tries to do everything.
- Use `LlmAgent` for flexible, reasoning-driven behavior and workflow agents (`SequentialAgent`, `ParallelAgent`, `LoopAgent`) for deterministic orchestration that doesn't need an LLM to decide the next step.
- Write instructions that define task boundaries, tool-use rules ("always call `lookup_order` before answering order questions"), and escalation behavior ("if you cannot resolve the issue, transfer to `human_handoff_agent`").
- Split multi-agent systems by responsibility (e.g., a "triage" agent, a "billing" agent, a "technical support" agent) rather than by implementation convenience.
- Keep the model name configurable (environment variable or config file) so it can be swapped across `gemini-2.0-flash`, `gemini-2.5-pro`, or other supported models without code changes.
- Use `output_schema` (a Pydantic model) when the agent's final response must be structured data consumed by other code.

## Tools

- Give tools narrow, typed inputs and outputs — a single function should do one thing, with a docstring the LLM uses to decide when to call it.
- Validate tool arguments before performing any side effect (writes, external calls, payments).
- Keep secrets, credentials, and privileged API keys out of agent instructions and prompts; inject them at the tool implementation layer instead.
- Handle tool errors explicitly and return actionable failure messages (e.g., `{"status": "error", "message": "order_id not found"}`) rather than raising unhandled exceptions that break the agent loop.
- Be aware of ADK tool limitations — some built-in tools (e.g., `google_search`, `code_execution`) cannot be combined with other tools on the same agent; delegate to a dedicated sub-agent instead.
- Prefer `FunctionTool` for custom Python logic, `AgentTool` to let one agent call another agent as a tool, and built-in tools (`google_search`, `BuiltInCodeExecutor`) only when their constraints fit the use case.

### Example: Defining an Agent with Tools

```python
from google.adk.agents import Agent
from google.adk.tools import FunctionTool
from google.adk.runners import InMemoryRunner
from google.genai import types


def get_order_status(order_id: str) -> dict:
    """Look up the current status of a customer order.

    Args:
        order_id: The unique identifier of the order, e.g. "ord-12345".

    Returns:
        A dict with 'status' ('found' or 'error') and either 'state'
        and 'updated_at', or an error 'message'.
    """
    if not order_id or not order_id.startswith("ord-"):
        return {"status": "error", "message": "invalid order_id format"}

    # In production this would call a real order service.
    orders = {"ord-12345": {"state": "shipped", "updated_at": "2026-08-30T10:00:00Z"}}
    order = orders.get(order_id)
    if order is None:
        return {"status": "error", "message": f"no order found for {order_id}"}
    return {"status": "found", **order}


root_agent = Agent(
    name="order_support_agent",
    model="gemini-2.0-flash",
    description="Answers customer questions about order status.",
    instruction=(
        "You help customers check their order status. "
        "Always call get_order_status before answering an order question. "
        "If the tool returns an error, apologize and ask the customer to "
        "double-check their order ID. Never invent an order status."
    ),
    tools=[FunctionTool(func=get_order_status)],
)


async def main() -> None:
    runner = InMemoryRunner(agent=root_agent, app_name="order_support")
    session = await runner.session_service.create_session(
        app_name="order_support", user_id="user-1"
    )
    message = types.Content(
        role="user", parts=[types.Part(text="What's the status of ord-12345?")]
    )
    async for event in runner.run_async(
        user_id="user-1", session_id=session.id, new_message=message
    ):
        if event.content and event.content.parts:
            for part in event.content.parts:
                if part.text:
                    print(part.text)


if __name__ == "__main__":
    import asyncio

    asyncio.run(main())
```

## Sessions, State, and Memory

- Use `session.state` for current-conversation data — user preferences discovered mid-conversation, counters, intermediate results.
- Use a `MemoryService` for cross-session recall and retrieval (facts that should persist after the session ends and be searchable in future sessions).
- Keep state values small and serializable (strings, numbers, small dicts/lists) — never store large files or binary payloads directly in session state.
- Make state keys stable and documented; use prefixes (`user:`, `app:`, `temp:`) to scope state lifetime consistently across the app.
- For production, use `DatabaseSessionService` or the managed Vertex AI session service instead of `InMemorySessionService`, which loses all state on process restart.

## Artifacts

- Use artifacts for generated files, uploaded files, reports, images, audio, and any other binary data — not session state.
- Configure an `ArtifactService` (e.g., `InMemoryArtifactService` for dev, `GcsArtifactService` for production) on the `Runner` before relying on artifact operations like `save_artifact` or `load_artifact`.
- Version artifact filenames intentionally (ADK auto-increments versions per filename) and avoid overwriting semantically different outputs under the same name.
- Store only references or short summaries in session state when the full content belongs in an artifact — e.g., keep `report_v2.pdf` as the artifact and `"generated report v2"` as the state note.

## Evaluation and Deployment

- Add `.evalset.json` test cases covering tool-call trajectories (which tools were called, in what order, with what arguments) and final response quality.
- Run `adk eval <agent_module> <eval_set_file>` in CI to catch prompt regressions before merging instruction changes.
- Use ADK's trace and event logs (`adk web`'s trace view, or `Runner` events) to debug agent decisions — inspect which tool was called, what arguments were passed, and what the model saw.
- Keep local development, staging, and production configuration separate (different `SessionService`/`ArtifactService` backends, different API keys, different model tiers).
- Add observability for latency, tool failures, token use, and handoff failures — wrap tool functions with timing/logging decorators and export metrics to your existing monitoring stack.
- Deploy to Vertex AI Agent Engine for a managed, autoscaled runtime, or containerize with `adk deploy cloud_run` / a custom Dockerfile for Cloud Run or GKE when more control over the environment is needed.

## Common Mistakes

- Making one agent responsible for every workflow instead of decomposing into focused sub-agents.
- Letting tools accept arbitrary shell, SQL, or HTTP input without validation, which turns a tool call into an injection vector.
- Relying on prompt text for access control — instructions like "never reveal other users' data" are guidance, not enforcement; check permissions in tool code.
- Hiding important side effects (payments, deletions, external notifications) behind generic tool names like `process` or `handle` — name tools for what they actually do.
- Forgetting to set `output_key` or `output_schema` when downstream code depends on a structured final response, leading to fragile string parsing.
- Skipping evals on instruction changes — a rephrased prompt can silently change which tools the model chooses to call.
