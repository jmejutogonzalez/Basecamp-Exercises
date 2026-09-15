# Developer Platform Build-Along — Learnings

Everything we learned working through [`Developer_Platform.ipynb`](Developer_Platform.ipynb): what each exercise asked for, a reference solution, the lessons behind it, and the Claude features it exercised. It should make sense without the notebook open.

---

## The scenario

**TechFlow** is a B2B SaaS company handling 500+ support tickets a day, at about 8 minutes each. It wants Claude to do Tier 1 triage: look up the ticket, search the knowledge base, decide on a fix, and hand back a machine-readable resolution.

We built that agent **directly on the Claude Messages API with the Python SDK, without a framework**, adding one capability per exercise:

```
tool schemas → agentic loop → structured output → adaptive thinking + effort → effort routing → streaming
```

- **Model:** Claude Sonnet 5 (`claude-sonnet-5`). The setup cell exposes it as `MODEL`, and also defines `FAST_MODEL` (Haiku 4.5) and `BIG_MODEL`. With a Bedrock key, the IDs get an `anthropic.` prefix, so always use the variables instead of hardcoding IDs.
- **Mock data:** 5 tickets (TKT-1042 to TKT-1046) and a small KB (KB-001 and up).
- **Local tool functions:** `get_ticket`, `search_kb`, `resolve_ticket`, dispatched by `execute_tool(name, input)`.

---

## Claude features we used

| Feature                     | API surface                                                               | Exercise     |
| --------------------------- | ------------------------------------------------------------------------- | ------------ |
| Tool use (client tools)     | `tools=[{name, description, input_schema}]`                               | 1, 2         |
| Agentic loop                | `stop_reason == "tool_use"`, `tool_use` → `tool_result` via `tool_use_id` | 2            |
| Structured outputs          | `output_config={"format": {"type": "json_schema", "schema": ...}}`        | 3            |
| Tool choice                 | `tool_choice={"type": "none"}`                                            | 3, 4, 6      |
| Adaptive thinking           | `thinking={"type": "adaptive"}`                                           | 2–6          |
| Thinking display            | `thinking={"type": "adaptive", "display": "summarized"}`                  | 4, 6         |
| Effort control              | `output_config={"effort": "low" \| "medium" \| "high"}`                   | 4, 5, 6      |
| Model tiering               | `FAST_MODEL` (Haiku 4.5) as a cheap triage classifier                     | 5            |
| Streaming                   | `client.messages.stream()`, event types, `get_final_message()`            | 6            |
| Message Batches _(planned)_ | `client.messages.batches.*`                                               | Extra Credit |

---

## Exercise 1 — Tool schemas

**Task:** `get_ticket` is given as an example. Write the `description` fields for `search_kb` and `resolve_ticket`, and the `status` enum.

```python
{
    "name": "search_kb",
    "description": "Search the TechFlow knowledge base for troubleshooting articles, procedures, and policies. "
                   "Use after reading a ticket to find the documented fix before resolving it.",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "A few specific keywords, e.g. 'duplicate refund' or 'webhook key rotation'"}
        },
        "required": ["query"]
    }
},
{
    "name": "resolve_ticket",
    "description": "Close out a ticket with the final resolution text and status. Call only after looking up the "
                   "ticket and searching the KB. Use 'escalated' when the escalation criteria apply.",
    "input_schema": {
        "type": "object",
        "properties": {
            "ticket_id": {"type": "string"},
            "resolution": {"type": "string"},
            "status": {"type": "string", "enum": ["resolved", "escalated", "pending"]}
        },
        "required": ["ticket_id", "resolution", "status"]
    }
}
```

**Lessons**

- **Claude picks tools by their descriptions.** Say _what_ a tool does, _when_ to use it, and where it fits in the sequence. A vague description leads to the wrong tool or the wrong order.
- **Parameter descriptions shape the arguments.** The mock `search_kb` returns any article containing _any_ word of 3+ letters from the query, up to 3 results. A long query matches nearly everything and returns the first three articles, so asking for "a few specific keywords" gets better results.
- **An `enum` limits a field to known values.** The exact values (`resolved`/`escalated`/`pending`) are our choice; the mock `resolve_ticket` stores whatever it receives.
- **Tools have side effects.** `resolve_ticket` changes the shared `TICKETS` dict, so rerunning a ticket overwrites its earlier status.

---

## Exercise 2 — The agentic loop (`run_agent`)

**Task:** fill the blanks. Loop while Claude wants tools, run them, send the results back, and call the API again.

```python
def run_agent(user_message: str):
    messages = [{"role": "user", "content": user_message}]
    params = dict(model=MODEL, max_tokens=32000, system=SYSTEM_PROMPT, tools=tools, thinking={"type": "adaptive"})

    response = client.messages.create(**params, messages=messages)

    while response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(result)})

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
        response = client.messages.create(**params, messages=messages)

    return response
```

**Lessons**

- **The API is stateless.** You send the whole conversation every call, so the loop _is_ the agent. Claude decides which tool comes next, and your code only runs tools and passes results back.
- **`stop_reason == "tool_use"`** means Claude wants tools run. Any other value (`end_turn`, `max_tokens`, `refusal`, …) ends the loop.
- **`tool_use_id=block.id`** links each result to its call. Return **all** results from one turn in **one** user message; splitting them teaches Claude to stop making parallel calls.
- **Append `response.content` as-is**, not just the text. It contains the thinking blocks, which must go back unchanged on the same model.
- **Parse `block.input` as data.** It's already a dict, so pass it straight to the function and don't match on its JSON text.

---

## Exercise 3 — Structured output (`run_agent_structured`)

**Task:** after the tool loop, make one final call that returns JSON matching `RESOLUTION_SCHEMA` (`diagnosis`, `solution_steps`, `confidence`, `escalation_needed`, `category`).

```python
    messages.append({"role": "user", "content": "Provide your structured resolution as JSON."})
    final = client.messages.create(
        model=MODEL, max_tokens=8000, system=SYSTEM_PROMPT,
        output_config={"format": RESOLUTION_SCHEMA},
        tool_choice={"type": "none"},
        thinking={"type": "adaptive"},
        messages=messages
    )
    return get_structured_result(final)   # json.loads of the LAST non-empty text block
```

**Lessons**

- **`output_config.format` guarantees JSON that fits the schema.** No regex parsing or "please reply in JSON" retries, so downstream systems (database, dashboard, escalation router) can trust the result.
- **Keep `format` out of the tool loop.** It constrains _all_ text output and gets in the way of tool use. Add it only on the final call.
- **`tool_choice={"type": "none"}`** stops Claude calling more tools on that final call, so it has to answer.
- **With thinking on, the response is `[thinking, text]`.** The JSON is in the **last** text block, which is why `get_structured_result` filters for `type == "text"`.
- **The schema needs `additionalProperties: False` and a `required` list.** Enums (`confidence`, `category`) make the output easy to route on.
- **Don't use assistant prefill** (starting the assistant's reply with `{`) to force JSON. Current models reject it; structured outputs replace it.

---

## Exercise 4 — Adaptive thinking + effort (`run_agent_thinking`)

**Task:** write the whole function. Run the tool loop with an `effort` level, print thinking blocks, then make the structured final call.

```python
def run_agent_thinking(user_message: str, effort: str = "high") -> dict:
    # "summarized" makes the thinking visible; on Sonnet 5 the default leaves it empty
    thinking = {"type": "adaptive", "display": "summarized"}

    messages = [{"role": "user", "content": user_message}]
    response = client.messages.create(
        model=MODEL, max_tokens=32000, system=SYSTEM_PROMPT, tools=tools,
        thinking=thinking, output_config={"effort": effort}, messages=messages
    )

    while response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "thinking":
                print(f"\n💭 Thinking:\n{block.thinking}\n")
            elif block.type == "tool_use":
                print(f"🔧 {block.name}({block.input})")
                result = execute_tool(block.name, block.input)
                tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(result)})

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
        response = client.messages.create(
            model=MODEL, max_tokens=32000, system=SYSTEM_PROMPT, tools=tools,
            thinking=thinking, output_config={"effort": effort}, messages=messages
        )

    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": "Provide your structured resolution as JSON."})
    final = client.messages.create(
        model=MODEL, max_tokens=16000, system=SYSTEM_PROMPT, tools=tools,
        thinking=thinking,
        output_config={"effort": effort, "format": RESOLUTION_SCHEMA},
        tool_choice={"type": "none"},
        messages=messages
    )
    return get_structured_result(final)
```

**Lessons**

- **Adaptive thinking:** Claude decides how much to reason on each turn, including _between_ tool calls: what to search for, whether a KB article fits, whether to escalate.
- **`effort` sets how much Claude thinks and how many tokens it uses** (`low` → `max`, default `high`). It sits inside `output_config` next to `format`. Low effort means fewer, more combined tool calls and shorter reasoning.
- **Thinking is hidden by default on Sonnet 5.** Without `"display": "summarized"`, thinking blocks arrive with empty text. Claude thinks, and you pay for it, either way; `display` only controls whether you see it. You never get the raw reasoning, only a summary.
- **Thinking traces make the agent auditable.** You can see _why_ it escalated or which article it trusted, which is exactly what TechFlow's support lead asked for.
- **Give the final call enough `max_tokens`.** At high effort, thinking uses part of the budget before the JSON; 16000 is safer than 8000.
- **Pass `tools` on the final call too**, because the history contains tool calls.
- **Thinking only allows `tool_choice` `auto` or `none`.** Forcing a tool (`any` or `{"type": "tool", ...}`) returns an error.
- **Budget-based thinking is gone.** `budget_tokens` returns a 400 on Sonnet 5; use `adaptive` plus `effort`.

---

## Exercise 5 — Routing effort by ambiguity _(our extension)_

**Goal:** instead of fixed high/low runs, give ambiguous tickets more thinking and clear ones less.

A side lesson: `_not_built_yet(fn_name, result)` can't do this. `fn_name` is only a label in the "isn't built yet" message; the guard just checks for `None`. Routing needs a function that _picks_ the effort.

### Option A — keyword rules (instant, free)

```python
AMBIGUITY_SIGNALS = ["intermittent", "random", "sometimes", "haven't changed", "no changes", "unclear", "not sure"]

def pick_effort(ticket_id: str) -> str:
    ticket = TICKETS[ticket_id]
    text = ticket["description"].lower()
    if any(signal in text for signal in AMBIGUITY_SIGNALS):
        return "high"      # unclear root cause
    if ticket["product_area"] == "api" or ticket["priority"] == "critical":
        return "medium"    # needs investigation or has high stakes
    return "low"           # clear cause, standard fix

def run_agent_auto(ticket_id: str) -> dict:
    effort = pick_effort(ticket_id)
    print(f"🧭 {ticket_id} → effort={effort}")
    result = run_agent_thinking(f"Resolve ticket {ticket_id}", effort=effort)
    if result is not None:
        result["effort"] = effort
    return result
```

| Ticket   | Issue                                         | Area / priority       | Effort |
| -------- | --------------------------------------------- | --------------------- | ------ |
| TKT-1042 | Charged twice ($4,500)                        | billing / high        | low    |
| TKT-1043 | Webhooks 401 after key rotation               | api / medium          | medium |
| TKT-1044 | Bulk export request                           | feature_request / low | low    |
| TKT-1045 | Admin locked out, 47 users blocked            | account / critical    | medium |
| TKT-1046 | Intermittent 500s, "haven't changed anything" | api / high            | high   |

### Option B — fast-model triage (judges meaning, not wording)

```python
AMBIGUITY_SCHEMA = {
    "type": "json_schema",
    "schema": {
        "type": "object",
        "properties": {
            "ambiguity": {"type": "string", "enum": ["low", "medium", "high"]},
            "reason": {"type": "string"}
        },
        "required": ["ambiguity", "reason"],
        "additionalProperties": False
    }
}

def classify_ambiguity(ticket_id: str) -> dict:
    response = client.messages.create(
        model=FAST_MODEL, max_tokens=1024,
        system="Rate how ambiguous this support ticket is: low (clear cause, standard fix), "
               "medium (likely cause, needs investigation), high (unclear root cause, intermittent, several explanations).",
        output_config={"format": AMBIGUITY_SCHEMA},   # no effort here — Haiku 4.5 rejects it
        messages=[{"role": "user", "content": get_ticket(ticket_id)}]
    )
    return get_structured_result(response)
```

**Lessons**

- **Use a cheap model to route and an expensive one to work.** A small Haiku call to classify costs little next to a high-effort Sonnet agent run.
- **Keyword rules are free but fragile**, since they miss ambiguous tickets worded differently. An LLM classifier handles that at the cost of one small call.
- **Check your assumptions against the data.** We first wrote `product_area == "technical"`, but the real areas are `billing`, `api`, `account`, `feature_request`. `technical` exists only as a resolution _category_.
- **Test the router alone first:** `{t: pick_effort(t) for t in TICKETS}` costs nothing.

---

## Exercise 6 — Streaming (`run_agent_streaming`)

**Task:** same agent, but thinking, tool calls and the final JSON appear live.

```python
def run_agent_streaming(user_message: str, effort: str = "high") -> dict:
    thinking = {"type": "adaptive", "display": "summarized"}
    messages = [{"role": "user", "content": user_message}]

    def stream_turn(**extra):
        with client.messages.stream(
            model=MODEL, max_tokens=32000, system=SYSTEM_PROMPT, tools=tools,
            thinking=thinking, messages=messages, **extra
        ) as stream:
            for event in stream:
                if event.type == "content_block_start":
                    block = event.content_block
                    if block.type == "thinking":
                        print("\n\n💭 Thinking: ", end="", flush=True)
                    elif block.type == "tool_use":
                        print(f"\n\n🔧 {block.name}(", end="", flush=True)
                    elif block.type == "text":
                        print("\n\n📝 ", end="", flush=True)
                elif event.type == "content_block_delta":
                    delta = event.delta
                    if delta.type == "thinking_delta":
                        print(delta.thinking, end="", flush=True)
                    elif delta.type == "text_delta":
                        print(delta.text, end="", flush=True)
                    elif delta.type == "input_json_delta":
                        print(delta.partial_json, end="", flush=True)
                elif event.type == "content_block_stop" and event.content_block.type == "tool_use":
                    print(")", end="", flush=True)
            return stream.get_final_message()

    response = stream_turn(output_config={"effort": effort})
    while response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(result)})
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
        response = stream_turn(output_config={"effort": effort})

    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": "Provide your structured resolution as JSON."})
    print("\n\n📋 Structured resolution:", end="", flush=True)
    final = stream_turn(output_config={"effort": effort, "format": RESOLUTION_SCHEMA},
                        tool_choice={"type": "none"})
    return get_structured_result(final)
```

**Lessons**

- **Streaming is about trust and UX.** A dashboard showing the agent reasoning and calling tools beats a 15–90 second spinner, and long high-effort calls avoid HTTP timeouts.
- **Event types to handle:**
  - `content_block_start`: a new block begins. Check `content_block.type`: `thinking`, `tool_use` or `text`.
  - `content_block_delta`: `thinking_delta` (`.thinking`), `text_delta` (`.text`), `input_json_delta` (`.partial_json`, the tool arguments arriving in pieces).
  - `content_block_stop`: the block is finished, and the SDK attaches it as `event.content_block`.
- **`stream.get_final_message()` returns the same `Message` as `create()`**, so the tool loop and `get_structured_result()` didn't change. Streaming is a display layer on top of the same logic.
- **One `stream_turn(**extra)` helper** serves the tool loop and the final structured call, which differ only in `output_config` / `tool_choice`.
- **`print(..., end="", flush=True)`**, or output is buffered and arrives in bursts.
- **Structured output streams too.** You watch the JSON build up, then parse it once at the end.
- **Full demo:** TKT-1045 (account lockout) combines everything: agentic loop, adaptive thinking, structured output and streaming.

---

## Extra Credit — how we'd approach it _(not implemented yet)_

Suggested order: **2 → 1 → 3**. #2 builds the quality checks that tell you whether #1 and #3 cost anything.

### 1. Tool choice controls

- **Already used:** `tool_choice={"type": "none"}` on every final structured call.
- **Extension, a tool-round limit:** after N rounds, pass `tool_choice={"type": "none"}` so the agent has to wrap up with what it has.
  ```python
  extra = {"tool_choice": {"type": "none"}} if round_num >= max_tool_rounds else {}
  response = client.messages.create(..., **extra)
  ```
- **Demo:** `none` from the first call means Claude never looks anything up and answers from the system prompt, which shows the risk of made-up answers.
- **Limit:** with thinking on, only `auto` and `none` work.

### 2. Effort optimization — quality/speed table

- **Quality needs expected answers**, or you only measure speed:
  - Category: `billing`→billing, `api`→technical, `account`→account, `feature_request`→feature_request.
  - Escalation: TKT-1045 should escalate (security, many users blocked). TKT-1042 shouldn't ($4,500 is under the $10k limit in KB-001 and the system prompt).
  - Optional: a `FAST_MODEL` judge scoring each resolution 1–5 against the KB article.
- **Cost:** add up `response.usage.input_tokens` / `output_tokens` across _every_ call in a run. Sonnet 5 costs **$2 input / $10 output per million tokens**.
- **Speed:** 5 tickets × 3 efforts = 15 agent runs, 10+ minutes one after another. Use `ThreadPoolExecutor(max_workers=5)` and turn off per-run printing.
- **Table** (via `tabulate`), one row per effort: category accuracy, escalation accuracy, average steps, average time, average output tokens, total cost.
- **One run per combination is noisy;** repeat 2–3× before trusting differences.
- **Hypothesis to test:** `low` handles the clear tickets for far fewer tokens, and `high` pays off mainly on TKT-1046. That would support the routing from Exercise 5.

### 3. Batch processing (50% off)

**A batch request is a single Messages call, so it can't run a tool loop.** Nothing runs your tools between turns.

| Approach                                  | How                                                                                                                                                    | Trade-off                                                                                                                               |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| **A. Fetch the data first** (recommended) | Call `get_ticket()` + `search_kb()` locally, put the results in the prompt, send one batch request per ticket with `output_config.format` and no tools | One batch, simple. It becomes a fixed workflow instead of an agent, which is fine because the tools read local data that doesn't change |
| **B. Batch round by round**               | Batch round 1 → run the tool calls locally → batch round 2 → …                                                                                         | Keeps the agent, but each round can take minutes (up to 24h). Complicated for little benefit                                            |

```python
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

requests = [
    Request(
        custom_id=tid,
        params=MessageCreateParamsNonStreaming(
            model=MODEL, max_tokens=16000, system=SYSTEM_PROMPT,
            thinking={"type": "adaptive"},
            output_config={"effort": "medium", "format": RESOLUTION_SCHEMA},
            messages=[{"role": "user", "content":
                f"Ticket:\n{get_ticket(tid)}\n\nKB results:\n{search_kb(query_for(tid))}\n\nResolve this ticket."}],
        ),
    )
    for tid in TICKETS
]
batch = client.messages.batches.create(requests=requests)
# poll client.messages.batches.retrieve(batch.id).processing_status until "ended", then:
results = {}
for r in client.messages.batches.results(batch.id):
    if r.result.type == "succeeded":
        results[r.custom_id] = get_structured_result(r.result.message)
```

- **Key results by `custom_id`**; they come back in any order.
- **`query_for(tid)` is a placeholder** for a short keyword query per ticket. Passing the whole description to the mock `search_kb` matches nearly every article (see Exercise 1).
- **Prompt caching stacks with the batch discount:** add `cache_control` to the shared system prompt.
- **Not available on Bedrock.** The Day 2 notebook skips live batch submission there.
- **Trade-off:** 50% cheaper, but no streaming, no agent choosing its own steps, and results take minutes to hours.

---

## Gotchas cheat sheet

| Symptom / trap                                       | Fix                                                           |
| ---------------------------------------------------- | ------------------------------------------------------------- |
| Thinking blocks print empty on Sonnet 5              | `thinking={"type": "adaptive", "display": "summarized"}`      |
| Tools stop working when JSON output is on            | Keep `format` out of the loop; add it only on the final call  |
| Agent keeps calling tools on the "give me JSON" call | `tool_choice={"type": "none"}`                                |
| JSON parse fails with thinking on                    | Read the **last** text block (`get_structured_result`)        |
| Final JSON cut off at high effort                    | Raise `max_tokens` on the final call (16000)                  |
| 400 on forced tool choice                            | Thinking only allows `auto` / `none`                          |
| 400 on `budget_tokens`                               | Use `adaptive` + `output_config.effort`                       |
| 400 passing `effort` to Haiku 4.5                    | Leave `effort` off for `FAST_MODEL`                           |
| Streamed output arrives in bursts                    | `print(..., end="", flush=True)`                              |
| Later turns lose context or break                    | Append the full `response.content`, not just text             |
| Router never picks "medium" for tech tickets         | Area is `api`, not `technical`                                |
| KB search returns the same articles for everything   | Use short, specific queries; the mock matches any shared word |

---

## Environment setup lessons

- **Python 3.10+ is required.** macOS's default `python3` was 3.9.6, so we built `.venv` from Homebrew's `python3.12` (`/opt/homebrew/bin/python3.12 -m venv .venv`).
- **Every notebook refuses to run on system Python.** Use the `.venv`, then pick it in VS Code: kernel picker → Select Another Kernel → Python Environments → `.venv`. If it's missing, run **Developer: Reload Window**.
- **API key:** in the gitignored `.env` at the repo root (`ANTHROPIC_API_KEY=sk-ant-...`), never in a cell. For Bedrock, use `AWS_BEARER_TOKEN_BEDROCK` + `AWS_REGION`.
- **SDK version:** `pip install -r requirements.txt` pulled `anthropic` 1.6.0; the requirements file was last verified on 0.104.x.
- **Running cells in VS Code:** **Shift+Enter** runs and moves to the next cell, **Ctrl+Enter** runs and stays (Cmd+Enter isn't reliable), or use **Run All**.
- **Agent cells are slow:** 30–90 seconds with thinking. `[*]` means the cell is still running.
