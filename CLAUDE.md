# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # then add GROQ_API_KEY
```

## Running the app

```bash
python app.py          # launches Gradio UI in browser
```

## Architecture

Plant Advisor is a two-tool conversational agent built with Gradio + Groq (llama-3.3-70b-versatile).

**Request flow:**

```
app.py (Gradio UI) → run_agent(user_message, history)
                           ↓
                     agent.py  ←→  Groq LLM (tool-calling loop)
                        ↓  ↓
              lookup_plant()   get_seasonal_conditions()
                    ↓                  ↓
             data/plants.json    data/seasons.json
```

**Key files and their roles:**

- `app.py` — Gradio UI, complete, do not modify. Calls `run_agent()` per turn.
- `agent.py` — Groq client, `TOOL_DEFINITIONS` (schemas for the LLM), `SYSTEM_PROMPT`, `dispatch_tool()` (routes LLM tool calls to Python functions), and `run_agent()` (the agent loop to implement).
- `tools.py` — `lookup_plant()` and `get_seasonal_conditions()` (data retrieval tools).
- `config.py` — `GROQ_API_KEY`, `LLM_MODEL`, `MAX_TOOL_ROUNDS`, `DATA_PATH`.
- `data/plants.json` — 15 plants keyed by slug (e.g. `"pothos"`, `"snake_plant"`). Each entry has `display_name`, `scientific_name`, `aliases`, and full care data.
- `data/seasons.json` — 4 seasons keyed by name. Each has watering/fertilizing/light/pest guidance.

## Tool-calling API pattern

The Groq API follows the OpenAI tool-calling convention. When the LLM wants a tool, `response.choices[0].message.tool_calls` is a non-empty list. Each item has:
- `.function.name` — tool name string
- `.function.arguments` — JSON string (may be `"null"` for no-arg tools)
- `.id` — must be echoed back as `tool_call_id` in the tool result message

The agent loop appends the assistant message **first**, then one `{"role": "tool", "tool_call_id": ..., "content": <json string>}` per tool call, then calls the LLM again. `MAX_TOOL_ROUNDS` caps the loop.

## What's left to implement

- `tools.py` → `lookup_plant()`: search `_plant_db` by slug key, `display_name`, `scientific_name`, then `aliases` (all case-insensitive). Return `{"found": True, "plant": <dict>}` or `{"found": False, "name": ..., "message": ...}`.
- `agent.py` → `run_agent()`: build messages list (system + history + user), loop calling the LLM and dispatching tool calls until no `tool_calls` or `MAX_TOOL_ROUNDS` exceeded, return final text.

`get_seasonal_conditions()` is already implemented — read it before implementing `lookup_plant`.
