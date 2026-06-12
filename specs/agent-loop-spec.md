# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `{"role": ..., "content": ...}` message dicts |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. The app creates its chat UI with
`type="messages"`, so Gradio history arrives as a list of API-format dicts with
`role` and `content` keys. Gradio may include extra keys (like `metadata`), so
copy only the two fields the API expects:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for msg in history:
    messages.append({"role": msg["role"], "content": msg["content"]})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it.

⚠️ For a no-argument tool call (like `get_seasonal_conditions` with no season),
the model may send `arguments` as the JSON string `"null"` — `json.loads` turns
that into `None`, not `{}`. Normalize before dispatching:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    raw_args = tool_call.function.arguments
    tool_args = json.loads(raw_args) if raw_args else {}
    if not isinstance(tool_args, dict):
        tool_args = {}
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
(a) No tool calls: after each LLM call check `if not assistant_message.tool_calls`.
    When falsy (None or empty list), return assistant_message.content immediately —
    this is the normal exit path.

(b) MAX_TOOL_ROUNDS reached: the for-loop exhausts its iterations without returning.
    After the loop, make one final LLM call WITHOUT tools so the model is forced to
    give a plain-text summary of everything it learned from the tool results already
    in the messages list. Return that content.
    (Alternative: return a static fallback string — but a final LLM call produces a
    much more useful response for the user.)
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
response.choices[0].message.content

`response.choices` is a list; index 0 is the first (and normally only) completion.
`.message` is the assistant message object.
`.content` is the plain string. Guard against None with:
    return assistant_message.content or "fallback string"
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: lookup_plant({"plant_name": "calathea"})
             ← returns full calathea care dict (difficulty: hard, etc.)
Round 1 tool call: get_seasonal_conditions({})   [may be batched in same round]
             ← returns current season data (e.g. Summer guidance)
Final response: personalized advice citing calathea's humidity needs,
               watering schedule, and summer-specific tips (heat + pest watch)
```

**What happens when you ask about a plant that isn't in the database?**

```
lookup_plant() returns {"found": False, "name": "string of pearls",
  "message": "No plant named 'string of pearls' was found ... Plants currently
  in the database: Pothos, Snake Plant, ..."}.
The LLM reads the message, tells the user the plant isn't in its database,
lists what IS available, and offers general succulent/trailing-plant advice
based on the user's description — exactly the graceful degradation behavior
specified in the system prompt.
```

**One thing about the tool call API that surprised you:**

```
A no-argument tool call (e.g. get_seasonal_conditions with no season) sends
arguments as the JSON string "null" rather than "{}" or an empty string.
json.loads("null") returns Python None, not an empty dict, which would crash
tool_args.get(...). The fix is an explicit isinstance check:
    if not isinstance(tool_args, dict): tool_args = {}
This is already handled in dispatch_tool() but must also be done before passing
to dispatch_tool() if you parse arguments yourself in the loop.
```
