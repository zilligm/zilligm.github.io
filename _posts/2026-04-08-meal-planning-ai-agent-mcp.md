---
layout: posts
title: "Building a Meal Planning AI Agent with MCP, Self-Hosted Apps, and a Local LLM"
date: 2026-04-08
published: true
description: "An AI agent that connects self-hosted Grocy and Mealie through MCP servers, powered by a local LLM, to answer: what can I cook this week with what I already have?"
---

I cook most of my meals at home. I also self-host [Grocy](https://grocy.info/) for pantry tracking and [Mealie](https://mealie.io/) for recipe management. Two separate apps, two separate databases, and no native way to ask a single question like: *"What can I cook this week with what I already have?"*

That question — and a desire to get hands-on with the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) — is what kicked off this project. I built an AI agent that connects to both services, reasons about ingredient availability, and produces weekly meal plans. It runs entirely on my local network, powered by a local LLM through Ollama.

<img src="/assets/images/meal_planner_agent/meal_planner_agent_diagram.png" alt="Meal planner agent architecture diagram" style="width: min(40%, 400px);">

## The Stack

The system has three layers:

1. **Two MCP servers** — one wrapping Grocy's REST API, one wrapping Mealie's. Each exposes a small set of tools over HTTP using [FastMCP](https://github.com/jlowin/fastmcp) with the `streamable-http` transport.
2. **A LangGraph agent** — connects to both MCP servers via `langchain-mcp-adapters`, discovers available tools at startup, and uses a ReAct loop to plan meals.
3. **A local LLM** — typically Gemma 3, Gemma 4, or Qwen, served by [Ollama](https://ollama.com/) on my home network.

Grocy and Mealie themselves run on my home server. The MCP servers run as Docker containers on a Raspberry Pi, with a GitLab CI/CD pipeline that builds images automatically and [Watchtower](https://containrrr.dev/watchtower/) handling rolling updates on the Pi. The agent runs locally on my desktop.

## What the MCP Servers Expose

Each MCP server is a thin Python service (FastMCP, `streamable-http` transport) that wraps the underlying app's REST API and returns structured Pydantic models.

**Mealie MCP**: 8 tools covering recipes, meal plans, and shopping lists:

- `list_recipes` / `get_recipe_ingredients` / `get_recipe_steps` — read the recipe catalog
- `add_recipe_mealplan` / `set_random_mealplan` — write to the meal plan
- `add_item_shopping_list` / `get_all_items_shopping_lists` — manage shopping lists
- `add_recipe_url` — import a recipe from the web

**Grocy MCP**: 2 tools covering pantry inventory:

- `list_products(filter, include_non_ingredients, days)` — query pantry stock with selectable filters (`in_stock`, `expiring`, `open`, `all`)
- `get_item_stock(item_id)` — detail for a single product

Both servers follow the same contract: every tool returns a typed success model or a `ToolError(code, message)` — never an unhandled exception. HTTP calls use session pooling with retry (3 retries, exponential backoff on 429/5xx). Structured JSON logging emits `request_id`, `tool_name`, and `duration_ms` on every call.

## The Grocy Aggregation Problem

Grocy tracks products at the barcode level. If I have three brands of olive oil, that's three separate product entries. But when a recipe calls for "olive oil," it means the generic ingredient.

The Grocy MCP server solves this by aggregating stock at the **parent-product level**. Grocy supports a parent/child product hierarchy, and the server merges quantities across all children before returning data. The agent never sees brand-level variants, just "Olive Oil: 2 bottles."

## How the Agent Works

The agent uses LangGraph's `create_react_agent`, which implements a ReAct (Reason + Act) loop. On startup, it connects to both MCP servers, discovers all available tools via `langchain-mcp-adapters`, and runs a preflight check (a lightweight call to each server) to catch configuration issues before the REPL starts.

The system prompt is where most of the behavioral tuning lives. It encodes rules like:

- **Tool results are the single source of truth.** The agent must never claim an ingredient exists unless a Grocy tool returned it in the current turn. Same for recipes.
- **Fuzzy semantic matching.** Mealie says "olive oil"; Grocy says "Extra Virgin Olive Oil." The LLM bridges that gap at inference time — no shared ingredient registry needed.
- **Filter selection logic.** "Do I have eggs?" → `list_products(filter="in_stock")`. "What's expiring?" → `list_products(filter="expiring")`. These rules are spelled out explicitly in the prompt because small models need the guidance.
- **Anti-hallucination guardrails.** Previous assistant messages are not evidence. If a fact wasn't verified by a tool call, the agent must say so.

The agent supports three modes: one-shot queries from the command line, an interactive REPL with bounded conversation memory, and a `--diagnostics` flag that prints all discovered tools grouped by server.

## The Hardest Part: Working with Small Local Models

Running the whole thing on a local LLM (4–12B parameters, quantized) has been the most persistent challenge. Cloud models like GPT-4 or Claude handle tool selection almost effortlessly. Local models need more scaffolding.

One concrete example: I originally had separate Grocy tools for each filter — `list_in_stock_products`, `list_expiring_products`, `list_open_products`, and so on. The LLM would routinely pick the wrong one, especially for queries like "what's in my pantry that expires soon?" where it had to choose between `list_in_stock_products` and `list_expiring_products`.

I consolidated everything into a single `list_products` tool with a `filter` parameter. One tool, one decision point, and a clear docstring explaining when to pick each filter value. That change alone noticeably improved the agent's accuracy.

More generally, I learned that with smaller models:

- **Fewer, broader tools beat many narrow ones.** The model struggles with a large tool catalog where the distinctions are subtle.
- **Explicit prompt rules matter.** Cloud models can infer "don't hallucinate" from context. Local models need the rule written out: *"An ingredient is available ONLY when it appears in an in_stock result with quantity > 0."*

## Deployment: CI/CD to a Raspberry Pi

The MCP servers are containerized with a straightforward `python:3.13-slim` base image and `uv` for dependency management. My GitLab server runs a CI pipeline that builds the Docker images and pushes them to a private registry. Watchtower, running on the Raspberry Pi, polls the registry and automatically pulls new versions.

This means I can push a fix to a MCP server and have it running in production within minutes, without SSH-ing into the Pi.

## What's Next

The system works for day-to-day meal planning, but there's room to grow:

- **More Grocy tools** — search by product name, manage open stock, health-check endpoints
- **Composite operations** — `check_recipe_feasibility` and `generate_shopping_list_for_mealplan` as server-side tools, reducing the number of agent round-trips
- **Better anti-hallucination** — a verified-facts layer that gates claims through deterministic checks before the LLM can surface them


## Resources

- Grocy: [grocy.info](https://grocy.info/)
- Mealie: [mealie.io](https://mealie.io/)
- FastMCP: [github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp)
- LangGraph: [github.com/langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)
- Model Context Protocol: [modelcontextprotocol.io](https://modelcontextprotocol.io/)
- Ollama: [ollama.com](https://ollama.com/)
