---
title: StackOne
description: Give a Pydantic AI agent actions on the user's SaaS accounts (HRIS, ATS, CRM, and more) via the StackOne integration platform.
---

# StackOne

Actions on the user's SaaS accounts (HRIS, ATS, CRM, and more) for Pydantic AI agents, via [StackOne](https://www.stackone.com).

[Source](https://github.com/pydantic/pydantic-ai-harness/tree/main/pydantic_ai_harness/stackone/) · [StackOne docs](https://docs.stackone.com)

## The problem it solves

Agents that act on business systems (list employees, update candidates, create tickets) need one integration per SaaS product: per-provider auth, per-provider APIs, per-provider schemas. StackOne exposes 10,000+ actions across 200+ providers through linked accounts and a single MCP endpoint, but wiring it up by hand means building auth headers, scoping requests to an account, and deciding how a large catalog should reach the model.

The `StackOne` capability connects an agent to one linked account's actions and handles the StackOne-specific parts: API-key auth, account scoping, action filtering, catalog delivery in either of StackOne's tool modes, and usage instructions. It is a thin composition of core toolsets (`MCPToolset` plus combinators), so it stays visible to anything that rewrites toolsets, such as durable execution wrappers.

## Installation

```bash
pip install "pydantic-ai-harness[stackone]"
```

Set `STACKONE_API_KEY` (or pass `api_key=`). Keys and linked accounts are managed at <https://app.stackone.com>.

## Usage

```python {test="skip"}
from pydantic_ai import Agent
from pydantic_ai_harness.stackone import StackOne

agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[
        StackOne(account_id='45320', actions=['*_list_*']),
    ],
)
result = agent.run_sync('List the first 5 employees')
print(result.output)
```

A linked account is one authenticated connection to one provider, selected per request with an `x-account-id` header. The lower-level `StackOneToolset` is also public, for use with [`Agent(toolsets=[...])`](/ai/tools-toolsets/toolsets/) or toolset combinators; it carries no instructions of its own.

## Filtering

`actions=['*_list_*', 'bamboohr_get_employee']` keeps tools whose full name matches any `fnmatch` glob (case-insensitive). Tool names follow `{connector}_{action}_{entity}`.

## Tool modes

StackOne serves its catalog in two modes, chosen with `tool_mode`:

| Mode | Tools registered | Best for |
|---|---|---|
| `individual` (default) | One tool per enabled action, each with its own schema | Catalogs up to a few hundred actions; per-tool validation, filtering, and approval |
| `search_execute` | Two meta-tools: search the catalog by natural-language query, execute an action by id | Very large catalogs; constant prompt footprint, no catalog fetch |

In `search_execute` mode individual action names never reach the agent, so `actions` globs cannot apply (construction warns), and per-tool wrappers like approval-by-name have to intercept the execute meta-tool instead. The injected instructions tell the model to search before executing; StackOne action ids are runtime values that must not be guessed.

For large catalogs in `individual` mode, Pydantic AI's [deferred tool loading](/ai/tools-toolsets/deferred-tools/) is the framework-side alternative: pass `defer_loading=True` (with a stable `id`) and tools stay out of the prompt until the model searches for them. Combining `defer_loading` with `search_execute` stacks one discovery hop on another; the capability warns about that too.

## How it composes

- **Code mode**: pass `code_mode=True` and every StackOne tool carries `code_mode=True` metadata, so [`CodeMode`](code-mode.md)`(tools={'code_mode': True})` batches StackOne calls inside one sandboxed `run_code`.
- **Approval**: wrap write actions with Pydantic AI's [tool approval](/ai/tools-toolsets/deferred-tools/#human-in-the-loop-tool-approval), e.g. `StackOneToolset(...).approval_required(...)` matching `*_create_*`, `*_update_*`, and `*_delete_*` names.
- **Several StackOne capabilities** on one agent (e.g. two accounts) need distinct `id`s; identical tool names across accounts surface as a tool-name conflict at run time.

## Agent spec (YAML/JSON)

All configuration is [spec](/ai/core-concepts/agent-spec/)-expressible; keep the API key in the `STACKONE_API_KEY` environment variable rather than in the file.

```yaml
capabilities:
  - type: StackOne
    account_id: '45320'
    actions: ['*_list_*']
```

```python {test="skip"}
from pydantic_ai import Agent
from pydantic_ai_harness.stackone import StackOne

agent = Agent.from_file('agent.yaml', custom_capability_types=[StackOne])
```

## Scope

This capability deliberately starts small: one explicit linked account per instance. Multi-account fan-out, account auto-discovery, and provider-based filters are candidate follow-ups.

The API may change while the capability stabilizes; breaking changes ship deprecation warnings where practical.

::: pydantic_ai_harness.stackone.StackOne

::: pydantic_ai_harness.stackone.StackOneToolset
