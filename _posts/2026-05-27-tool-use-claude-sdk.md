---
layout: post
title: "Tool Use Rings: How Claude Actually Calls Tools (Under the Hood)"
date: 2026-05-27
categories: [ai-engineering, anthropic, tool-use, agents]
tags: [anthropic, claude, tool-use, mcp, agentic-loop, python-sdk, ai-engineering]
---

I built a **GitHub Repo Health Checker** that demonstrates five tool-use patterns using the Anthropic Python SDK. Point it at any public GitHub repo and it runs a full health check across all five patterns. This post walks through every layer: the tools themselves, how Claude decides to call them, how tool responses flow back to the API, how the conversation ends, and what each ring actually demonstrates.

```bash
python demo.py --repo torvalds/linux   # works on any public repo
```

---

## What "tool use" actually means at the API level

When you call `client.messages.create()` with a `tools=` parameter, you're not telling Claude to *run* anything. You're giving the model a menu. Claude reads the tool schemas alongside your message and decides — during inference — whether calling a tool will help it answer better.

If it does, the response comes back with `stop_reason="tool_use"` and a `tool_use` content block:

```json
{
  "type": "tool_use",
  "id": "toolu_01Abc...",
  "name": "get_github_repo",
  "input": { "owner": "anthropics", "repo": "anthropic-sdk-python" }
}
```

Claude stops. It waits. **You** run the tool. You send back a `tool_result`:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01Abc...",
  "content": "{\"stars\": 4821, \"language\": \"Python\", ...}",
  "is_error": false
}
```

Claude resumes with the real data and produces its final answer (`stop_reason="end_turn"`). The "AI calling tools" you hear about is a **loop between the model and your application code**. The model decides *what* to call; you decide *how* to execute it.

---

## The tools in this project

There are six tools registered across the five rings. Each is a client-side Python function backed by a real API call.

### GitHub tools (4 tools)

All hit the GitHub REST API — no auth token required for public repos (60 req/hr unauthenticated).

| Tool | What it fetches |
|------|----------------|
| `get_github_repo` | Stars, forks, open issue count, language, description, last push |
| `get_github_issues` | Most recent open issues with titles, labels, and dates |
| `get_github_prs` | Most recent open pull requests with author and date |
| `get_github_contributors` | Top contributors by commit count |

Each returns a clean dict; the dispatcher serializes it to JSON and returns `(json_string, is_error=False)`. On an HTTP error (e.g. 404), it returns `(error_json, is_error=True)` — the same shape, just flagged so Claude knows to reason about the failure.

### `web_search`

Hits the DuckDuckGo Instant Answer API — free, no key required. Returns an abstract and up to 3 related topics. Works well for famous entities and definitions; returns "No instant answer found" for niche queries like specific GitHub repos. In a production system this would be swapped for Tavily or Brave Search.

### `execute_python`

Runs arbitrary Python code in a subprocess with a 10-second timeout. Used in Rings 4 and 5 to do math on numbers Claude has already fetched — percentage of bug-labeled issues, ratio of stars to forks. Keeps arithmetic deterministic rather than relying on the model to compute it.

### Which ring uses which tools

| Ring | Tools available to Claude |
|------|--------------------------|
| Ring 1 | `get_github_repo` |
| Ring 2 | `get_github_repo`, `get_github_issues`, `get_github_prs` |
| Ring 3 | `get_github_repo`, `get_github_contributors`, `web_search` |
| Ring 4 | `get_github_repo`, `get_github_issues`, `execute_python` |
| Ring 5 | `get_github_repo`, `web_search`, `execute_python` |

Claude only sees the tools registered for each ring. If `web_search` isn't in the schema list for a given ring, Claude has no way to call it — the schema IS the permission boundary.

---

## Client-side vs Server-side (MCP) tools

**Client-side tools** are Python functions you define and run in your own process. When Claude emits `tool_use`, your code catches it, calls the function, and sends the result back. All six tools in this project are client-side.

**MCP server tools** are tools hosted in a separate process (or remote server) that Claude connects to via the [Model Context Protocol](https://modelcontextprotocol.io). The SDK discovers the tool list automatically via `tools/list` RPC and routes `tool_use` blocks to the server. Claude has no idea whether a tool is client-side or server-side — the `tool_use` block looks identical either way.

```
Client-side:                          MCP server-side:
┌──────────┐  tool_use   ┌─────────┐   ┌──────────┐  tool_use   ┌─────────┐  stdio/SSE  ┌───────────┐
│  Claude  │ ──────────► │  Your   │   │  Claude  │ ──────────► │   SDK   │ ──────────► │  MCP      │
│  (API)   │ ◄────────── │  code   │   │  (API)   │ ◄────────── │  proxy  │ ◄────────── │  Server   │
└──────────┘  tool_result└─────────┘   └──────────┘  tool_result└─────────┘             └───────────┘
              (you execute it here)                               (server executes it here)
```

---

## The five rings

### Ring 1 — Single Tool, Single Run

**Tools used:** `get_github_repo`

The foundational pattern. One tool declared, one tool_use emitted, one execution, one final answer.

```python
# Step 1: send message with one tool on the menu
response = client.messages.create(
    model=MODEL,
    max_tokens=512,
    tools=schemas_for("get_github_repo"),
    messages=[{"role": "user", "content": "Star count and language of anthropics/anthropic-sdk-python?"}],
)
# response.stop_reason == "tool_use"

# Step 2: execute the tool
tool_block = next(b for b in response.content if b.type == "tool_use")
result_json, is_error = execute_tool(tool_block.name, tool_block.input)

# Step 3: send tool_result back, get the final answer
final = client.messages.create(
    model=MODEL,
    max_tokens=512,
    tools=schemas_for("get_github_repo"),
    messages=[
        {"role": "user",      "content": original_message},
        {"role": "assistant", "content": response.content},       # ← Claude's tool_use block
        {"role": "user",      "content": [{
            "type":        "tool_result",
            "tool_use_id": tool_block.id,
            "content":     result_json,
            "is_error":    is_error,
        }]},
    ],
)
# final.stop_reason == "end_turn"
answer = next(b.text for b in final.content if b.type == "text")
```

The `tool_use` block Claude emitted and the `tool_result` you returned both become part of the conversation history. Claude "remembers" what it fetched because it's all in the messages list — there's no hidden state.

---

### Ring 2 — Agentic Loop

**Tools used:** `get_github_repo`, `get_github_issues`, `get_github_prs`

Give Claude a compound question — repo stats, recent issues, and open PRs. It calls tools one or more times per iteration until it has everything it needs, then stops with `end_turn`.

```python
MAX_ITERATIONS = 10
messages = [{"role": "user", "content": compound_question}]

while True:
    response = client.messages.create(
        model=MODEL, max_tokens=1024,
        tools=tools, messages=messages
    )
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason == "end_turn":
        return next(b.text for b in response.content if b.type == "text")

    tool_results = []
    for block in response.content:
        if block.type != "tool_use":
            continue
        result_json, is_error = execute_tool(block.name, block.input)
        tool_results.append({
            "type":        "tool_result",
            "tool_use_id": block.id,
            "content":     result_json,
            "is_error":    is_error,
        })
    messages.append({"role": "user", "content": tool_results})
```

Each iteration Claude reads the entire conversation history — including all tool results accumulated so far — and reasons about what it still needs. There's no separate planner. **The model IS the planner.**

---

### Ring 3 — Parallel Multi-Tool Run

**Tools used:** `get_github_repo`, `get_github_contributors`, `web_search`

When Claude sees multiple independent data needs in one prompt, it returns **multiple `tool_use` blocks in the same response**. We execute them concurrently with `ThreadPoolExecutor` and return all results in a single user turn.

```python
# Claude returns this in a single response:
response.content == [
    ToolUseBlock(name="get_github_repo",         input={...}),
    ToolUseBlock(name="get_github_contributors", input={...}),
    ToolUseBlock(name="web_search",              input={...}),
]

# Execute all three concurrently:
with ThreadPoolExecutor(max_workers=len(tool_uses)) as pool:
    futures = {pool.submit(execute_tool, b.name, b.input): b for b in tool_uses}
    for future in as_completed(futures):
        block_id, (result_json, is_error) = future.result()
        results[block_id] = (result_json, is_error)

# Return all results in one user turn:
messages.append({
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": tid, "content": res, "is_error": err}
        for tid, (res, err) in results.items()
    ]
})
```

The model decides fan-out, not you. Prompt words like *"simultaneously"* and *"all three"* signal independence to the inference engine. This is a meaningful latency win when tools have non-trivial network round-trips.

---

### Ring 4 — Error Handling

**Tools used:** `get_github_repo`, `get_github_issues`, `execute_python`

Tools fail. The `is_error=True` flag in `tool_result` is how you tell Claude a tool failed without crashing the conversation. Claude receives the error as context and reasons about whether to continue, fall back, or report partial results.

This ring deliberately injects a 404 for a non-existent repo alongside a real one. Claude fetches data for the real repo (succeeds), gets a 404 for the fake one (fails), then uses `execute_python` to compute what percentage of the real repo's issues are labeled "bug":

```python
# Forced 404 for the non-existent repo:
if block.input.get("repo") == "this-repo-does-not-exist-xyz":
    result_json = json.dumps({"error": "404 Not Found", "status_code": 404})
    is_error = True

# The real dispatcher handles real HTTP errors the same way:
except requests.HTTPError as e:
    status = e.response.status_code
    return json.dumps({"error": f"GitHub API {status}", "status_code": status}), True
```

Claude's response acknowledges the failure and delivers a useful partial answer rather than stopping. `is_error=True` is not a hack — it's a first-class design concept. Claude is trained to reason about tool failures as information.

---

### Ring 5 — Beta SDK ToolRunner Abstraction

**Tools used:** `get_github_repo`, `web_search`, `execute_python`

Two ideas in one ring:

**A) `client.beta.messages.create()`** — The `token-efficient-tools-2025-02-19` beta sends tool schemas in a more compact wire format, reducing token consumption by ~40% when you have many tools declared. Same behaviour, lower cost.

**B) `ToolRunner` class** — Encapsulates the entire agentic loop (Ring 2) behind a clean interface. Callers never see `messages[]`, `stop_reason` checks, or `tool_result` construction.

```python
class ToolRunner:
    def run(self, prompt: str) -> str:
        messages = [{"role": "user", "content": prompt}]
        for i in range(self.max_iterations):
            response = self.client.beta.messages.create(
                model=self.model, max_tokens=self.max_tokens,
                tools=self.tools, messages=messages,
                betas=["token-efficient-tools-2025-02-19"],   # ← compact schema format
            )
            messages.append({"role": "assistant", "content": response.content})
            if response.stop_reason == "end_turn":
                return next(b.text for b in response.content if b.type == "text")
            # execute tools, append results, loop...
```

The call site becomes:

```python
runner = ToolRunner.for_research()
answer = runner.run(f"Full health report on {repo}.")
```

Adding a new tool is a one-liner in `tools/__init__.py`. The loop, error handling, and beta header are all inherited automatically.

---

## How the tool response flows back to Claude

Every tool result — success or failure — follows the same path:

```
1. Claude emits tool_use block       → stop_reason="tool_use"
2. Your code calls execute_tool()    → (result_json, is_error)
3. You append to messages:
     {"role": "assistant", "content": [tool_use_block, ...]}
     {"role": "user",      "content": [{"type": "tool_result",
                                        "tool_use_id": ...,
                                        "content": result_json,
                                        "is_error": is_error}]}
4. You call messages.create() again  → Claude sees the result as context
5. If Claude needs more data         → go to step 1
   If Claude is done                 → stop_reason="end_turn", extract text
```

The conversation history is the memory. There is no hidden state — Claude knows what it already fetched because the tool_use block and tool_result are both in the messages list it receives each time.

---

## How the inference engine decides to call tools

Claude's tool-calling behaviour emerges from **how tools are described in the schema**, not from hard-coded rules. The description is injected into the prompt in a special XML format the model was trained on. Write it like documentation for a careful engineer — vague descriptions produce unpredictable calls, precise ones produce reliable agents.

The **`tool_choice` parameter** lets you override the default:
- `{"type": "auto"}` — Claude decides (default)
- `{"type": "any"}` — Claude must call a tool; you choose which ones are available
- `{"type": "tool", "name": "get_github_repo"}` — Claude must call this specific tool

---

## Running it

```bash
cd phase-1-tool-use-rings
pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-...

python demo.py                              # all 5 rings, default repo
python demo.py --repo torvalds/linux        # any public GitHub repo
python demo.py --ring 3                     # parallel tools only
python demo.py --ring 3 --repo pallets/flask
python demo.py --quiet                      # final answers only

pytest tests/ -v                            # 18 unit tests, no API key needed
```

The `--repo` flag accepts any public GitHub repo in `OWNER/REPO` format. No GitHub token required — the public REST API allows 60 unauthenticated requests per hour.

---

## Key takeaways

The model doesn't "call" anything. It generates structured JSON that looks like a function call. Your code runs the function and returns the result. Understanding this loop is what separates engineers who build reliable agents from those who treat the model as a black box.

Tool descriptions are decision logic. The model matches them against its training to decide when to call what. Write descriptions precisely — this is where agent reliability is won or lost.

Parallel tool calls are the model being efficient, not a feature you toggle. Design your prompts to make data independence obvious and fan-out happens automatically.

`is_error=True` is intentional design. Claude reasons about tool failures as data. Use the flag honestly instead of masking errors inside successful-looking responses.

The `ToolRunner` abstraction matters at scale. Rings 1–4 are the physics of tool use. Ring 5 is the engineering discipline of not repeating the same loop everywhere.

---

*Phase 0 deliverable: [MCP Permission Filter](/2026/05/25/phase-0-mcp-permission-filter.html)*  
*Full source: [phase-1-tool-use-rings](https://github.com/jainudi48/ai-engineering-journey/tree/main/phase-1-tool-use-rings)*
