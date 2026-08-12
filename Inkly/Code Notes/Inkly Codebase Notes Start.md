# Inkly — Codebase Overview

## Key Concepts

| Concept | What It Is |
|---|---|
| **Ollama** | A tool for running LLMs locally or on a server. Inkly does not build its own AI — it calls Ollama the same way a website calls a weather API. |
| **Plugins** | Modules that each collect one specific type of cluster information. They run live Slurm commands and return text. Each one checks one thing. |
| **Retrieval** | Instead of running every plugin for every question, Inkly figures out which plugins are relevant by converting both the question and plugin descriptions into numbers (embeddings) and finding the closest matches. |
| **Conversation history** | Saved in `~/.inkly/conversations/`. Lets follow-up questions like "What about the GPU partition?" work correctly — Inkly remembers what was said before. |
| **SQLite database** | A simple file-based database (`~/.inkly/jobs.db`) that stores historical job data. Lets Inkly answer data-driven questions using real past outcomes rather than guessing. |

## `inkly/ink_core.py` — Entry Point

This is the first file that runs when you type `ink <question>`. It runs in 5 steps:

### Step 1 — Build the query
Gets the user's question from the command line arguments. If the user typed `ink` with nothing after it, prints a usage message and exits with code `1` (standard way to signal an error).

### Step 2 — Load the config
```python
cfg = _load_config()
```
Reads `~/.inkly/config.toml` from disk. Wrapped in a `try/except` so that a missing or malformed config file prints a clean error instead of a long Python traceback.

### Step 3 — Initialize the runtime and run the query
```python
runtime = InklyRuntime(cfg)
user_id = _build_user_id()
response = runtime.handle_query(user_id, query)
```
This is where all the real work happens — plugins run, the retriever picks what's relevant, the prompt is assembled, and the AI is called. All of that complexity lives inside `InklyRuntime`.

**Next to explore:** `inkly/core/runtime.py` — where `InklyRuntime` and `handle_query` are defined.

### Step 4 — Print the response
```python
if not sys.stdout.isatty():
    print(response)
elif response and not response.endswith("\n"):
    print()
```
`sys.stdout.isatty()` checks whether output is going to an interactive terminal or being piped somewhere. If interactive, the backend already streamed the response character-by-character while generating — printing again would duplicate it. If piped, it prints the full response now.

### Step 5 — Exit
Returns `0` (success) or `1` (failure). `raise SystemExit(main())` at the bottom converts this into the process exit code.

## Separation of Concerns

`ink_core.py` is intentionally short. It only handles the entry point: receive input, set up the big pieces, hand off, return output. All complex logic lives in other files. This pattern repeats throughout the codebase.

---

## Related Notes

- [[ink_core.py]] — first file to explore in depth
- [[runtime.py]] — the central pipeline
- [[llm_backend.py]] — how Ollama is called
- [[Plugin System]] — how plugins are discovered and run
- [[retriever.py]] — how relevant plugins are selected
