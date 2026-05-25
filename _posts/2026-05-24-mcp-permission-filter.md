---
layout: post
title: "MCP Permission Filter: My First Real Python + Pydantic Project"
date: 2026-05-24
categories: [ai-engineering, python, pydantic, mcp]
tags: [python, pydantic, mcp, permissions, rbac, ai-engineering]
---

I'm 90 days into a self-designed AI Engineering preparation roadmap. Phase 0 is the Python Survival Kit (Days 1–4) — not a tutorial, but a real deliverable. Here's what I built and what I actually learned.

## The Deliverable: MCP Permission Filter

Modern AI systems connect to dozens of **MCP servers** — GitHub, Slack, Notion, Jira, databases, internal APIs. Each server exposes tools (think: callable functions). The problem is that not every user should see or call every tool. A contractor shouldn't be able to delete your GitHub repo. A read-only analyst shouldn't post to Slack.

I built a permission filter that solves exactly this: given a user and a list of MCP servers, return only the tools that user is authorized to call.

## The Architecture

Three Pydantic models do all the heavy lifting:

```python
class Permission(str, Enum):
    READ  = "read"
    WRITE = "write"
    ADMIN = "admin"

class MCPTool(BaseModel):
    name: str
    description: str
    required_permission: Permission

class MCPServer(BaseModel):
    server_id: str
    display_name: str
    tools: list[MCPTool]

class User(BaseModel):
    user_id: str
    name: str
    server_permissions: dict[str, Permission]  # server_id -> their level
```

The core insight is a **permission hierarchy**: admin ⊇ write ⊇ read. A user with `write` access can call any `read` tool too. This is modeled with a simple lookup:

```python
PERMISSION_HIERARCHY: dict[Permission, set[Permission]] = {
    Permission.READ:  {Permission.READ},
    Permission.WRITE: {Permission.READ, Permission.WRITE},
    Permission.ADMIN: {Permission.READ, Permission.WRITE, Permission.ADMIN},
}

def satisfies(user_permission: Permission, required: Permission) -> bool:
    return required in PERMISSION_HIERARCHY[user_permission]
```

## The Filter Function

The main function is refreshingly simple once the models are right:

```python
def filter_mcp_tools(user: User, servers: list[MCPServer]) -> AuthorizedToolView:
    authorized: dict[str, list] = {}
    for server in servers:
        allowed_tools = [
            tool for tool in server.tools
            if user.can_use_tool(server.server_id, tool)
        ]
        if allowed_tools:
            authorized[server.server_id] = allowed_tools
    return AuthorizedToolView(
        user_id=user.user_id,
        user_name=user.name,
        authorized_tools=authorized,
    )
```

Servers the user has zero access to are omitted entirely from the output — so the AI agent never even knows they exist.

## Real Output

Here's Dave, an external contractor who only has Jira write access. After filtering across 4 MCP servers (GitHub, Slack, Notion, Jira), he sees:

```
User: Dave (External Contractor) (dave)

  [jira]
    • list_issues [read] — List issues in a project
    • get_issue [read] — Get issue details
    • create_issue [write] — Create a new ticket
    • transition_issue [write] — Move issue through workflow

  Total: 4 tool(s) authorized
```

No GitHub. No Slack. No Notion. Clean.

## What I Actually Learned

**Pydantic v2 is production-grade.** The `model_validator(mode="after")` hook to catch duplicate tool names, `Field(...)` for required fields with descriptions, `frozen=True` for immutable value objects — these aren't tutorials tricks. They're the real patterns used in production AI systems.

**`str, Enum` is the right choice for typed string constants.** Because `Permission` inherits from `str`, it serializes cleanly to JSON and plays well with APIs without any extra conversion.

**Type hints aren't just documentation.** With Pydantic, they're runtime validation. Passing `"superuser"` as a permission value raises a `ValidationError` immediately — before it causes a silent bug at 2am.

**The filter logic itself is trivial when the models are right.** I spent 80% of my time on the data model and 20% on the actual function. This is exactly how it should be.

## Tests

25 tests, all passing. The test classes mirror the architecture:
- `TestPermissionHierarchy` — 7 tests covering the hierarchy edge cases
- `TestUserCanUseTool` — 6 tests covering user ↔ tool authorization
- `TestFilterMCPTools` — 7 tests including empty servers, no-access users, partial server access
- `TestBatchFilter` — 2 tests for the batch variant
- `TestModelValidation` — 3 tests for Pydantic validation behavior

## What's Next

Phase 1 is **FastAPI + LLM Gateway** (Days 5–12). I'll be wrapping this permission filter behind a real HTTP API, adding auth middleware, rate limiting, and streaming responses from Claude. The permission filter will become a middleware layer that intercepts tool calls before they reach the MCP servers.

The full code is on [GitHub](https://github.com/jainudi48/ai-engineering-journey).

---

*Part of my 90-day AI Engineering preparation roadmap. Following along? Each phase ends with a working deliverable, a GitHub commit, and a post like this one.*
