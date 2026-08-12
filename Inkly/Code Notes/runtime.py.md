# `runtime.py` — Detailed Walkthrough

**File location:** `inkly/core/runtime.py`  
**Role:** The central pipeline. Wires all the pieces together and handles every query end-to-end.

---

## What This File Does

`ink_core.py` hands off to `InklyRuntime` with a single call: `runtime.handle_query(user_id, query)`. Everything that happens between receiving a question and returning an answer lives here.

---

## Imports and Setup

```python
from inkly.core.conversation import ConversationManager
from inkly.llm.backend import LLMBackend
from inkly.plugins.manager import PluginManager
from inkly.retrieval.retriever import PluginRetriever
```

The runtime imports the four main subsystems it coordinates. It doesn't do the work of any of them — it just calls them in the right order. This is the **orchestrator** pattern.

---

## `InklyRuntime.__init__()` — Initialization

```python
self.conversation = ConversationManager(config)
self.plugins = PluginManager()
self.backend = LLMBackend(config)
self.retriever = None
self._request_gate = BoundedSemaphore(value=self.config.core.max_concurrent_requests)
```

When the runtime is created, it sets up all four pieces:

| Field | What It Is |
|---|---|
| `self.conversation` | Tracks per-user conversation history |
| `self.plugins` | Discovers and runs plugins |
| `self.backend` | Sends prompts to Ollama, gets responses |
| `self.retriever` | Plugin selector — starts as `None` (built on first use) |
| `self._request_gate` | A semaphore — limits how many requests can run at once |

A **semaphore** is a concurrency tool. `BoundedSemaphore(value=4)` means at most 4 requests can be processed simultaneously. The `with self._request_gate:` block at the start of `handle_query` claims one slot and releases it automatically when done, even if something goes wrong.

---

## `BASE_RESPONSE_CONTRACT` — The System Prompt

```python
BASE_RESPONSE_CONTRACT = textwrap.dedent("""
You are Inkly, an HPC assistant for a Slurm-based cluster.
Rules:
- Plain text only.
- Use the provided plugin context and conversation history when relevant.
- Do not invent cluster state, commands, files, or paths.
...
""").strip()
```

This is the **system prompt** — instructions sent to the AI at the start of every single request, before anything else. It tells the model who it is and how to behave. `textwrap.dedent()` removes the leading indentation from the string so it formats cleanly.

---

## Prompt Assembly — Four Sections

The full prompt sent to the AI is built from four labeled sections, always in this order:

```
=== INKLY RESPONSE CONTRACT ===
[rules for the model]

=== CONVERSATION HISTORY ===
[past turns, if any]

=== PLUGIN CONTEXT ===
[live cluster data from plugins]

=== USER QUERY ===
[the actual question]
```

Each section is built by its own method, and empty sections are automatically skipped. This is clean design — each method has one job and the final `assemble_prompt()` just joins them.

```python
sections = [
    self._build_contract_section(),
    self._build_history_section(history_lines),
    self._build_plugin_section(plugin_outputs),
    self._build_query_section(query),
]
non_empty_sections = [s for s in sections if s.strip()]
prompt = "\n\n".join(non_empty_sections)
```

If the assembled prompt is too long, it gets **truncated from the front** (keeping the most recent content):
```python
if len(prompt) > self.config.core.max_prompt_length:
    prompt = prompt[-self.config.core.max_prompt_length:]
```

---

## `handle_query()` — The Full Pipeline

This is the method `ink_core.py` calls. Here is the step-by-step flow:

### Step 1 — Record the user's question
```python
self.conversation.append_turn(user_id, "user", query)
```
The question is saved to conversation history before anything else, so follow-up questions will have context even if something later fails.

### Step 2 — Discover available plugins
```python
discovered = self.plugins.discover()
```
Asks the plugin manager to scan the `inkly/plugins/` folder and return all valid plugins.

### Step 3 — Select which plugins to run
This is where retrieval comes in. If retrieval is enabled in config, `PluginRetriever` scores all plugins against the user's query and returns only the most relevant ones. If retrieval fails or is disabled, it falls back to running **all** plugins.

```python
if retrieval_enabled:
    selected_plugins = list(retriever.select_plugins(query, discovered))
else:
    selected_plugins = list(discovered.values())
```

### Step 4 — Run the selected plugins
```python
for plugin in selected_plugins:
    plugin_outputs[plugin.name] = plugin.run()
```
Each plugin is called and its output (a string of formatted text) is stored. If a plugin crashes, the error is captured as a string rather than crashing the whole request.

### Step 5 — Build conversation history context
```python
history_lines = self.conversation.build_context(user_id, current_query=query, ...)
```
Fetches recent conversation turns to include in the prompt. Older turns may be summarized to save space.

### Step 6 — Assemble the prompt
```python
prompt = self.assemble_prompt(query=query, history_lines=history_lines, plugin_outputs=plugin_outputs)
```
Combines all four sections into the final string that gets sent to the AI.

### Step 7 — Call the AI
```python
response = self.backend.generate(prompt)
```
Sends the prompt to Ollama and waits for a response.

### Step 8 — Save the response and return
```python
self.conversation.append_turn(user_id, "assistant", response)
return response
```
The AI's answer is saved to conversation history (so future questions can reference it), then returned to `ink_core.py`.

---

## The Big Picture

```
handle_query()
    │
    ├── conversation.append_turn()     — save user question
    ├── plugins.discover()             — find all plugins
    ├── retriever.select_plugins()     — pick relevant ones
    ├── plugin.run() × N               — collect live cluster data
    ├── conversation.build_context()   — get history for prompt
    ├── assemble_prompt()              — build the full prompt
    ├── backend.generate()             — call the AI
    └── conversation.append_turn()     — save AI answer
```

---

## Next Files to Explore

- `inkly/llm/backend.py` — what `backend.generate()` actually does (sending to Ollama)
- `inkly/plugins/manager.py` + `queue_status.py` — how plugins are discovered and what one looks like
- `inkly/retrieval/retriever.py` — how plugins are scored and selected

---

## Related Notes

- [[ink_core.py]] — the entry point that calls `handle_query`
- [[llm_backend.py]] — what happens inside `backend.generate()`
- [[Plugin System]] — how `PluginManager` and plugins work
- [[retriever.py]] — how plugins are scored and selected
- [[Testing Inkly]] — how the runtime pipeline is tested
