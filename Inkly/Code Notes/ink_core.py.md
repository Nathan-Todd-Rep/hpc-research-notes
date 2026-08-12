# `ink_core.py` 

**File location:** `inkly/ink_core.py`  
**Role:** Main CLI entry point. Wires the whole system together.

---

## Imports (lines 1–8)

```python
import os
import sys
from pathlib import Path

from inkly.config import TomlParser
from inkly.core.runtime import InklyRuntime
```

- `os` — reads environment variables (e.g. who the current user is).
- `sys` — accesses command-line arguments (`sys.argv`) and lets you write to stderr.
- `Path` — a modern Python way to work with file paths. The `/` operator on a `Path` object joins path segments (not division).
- `TomlParser` and `InklyRuntime` — imported from within the Inkly project itself.

---

## Path Constants (lines 13–16)

```python
DEFAULT_INKLY_HOME = Path.home() / ".inkly"
CONFIG_PATH = DEFAULT_INKLY_HOME / "config.toml"
```

`Path.home()` returns the current user's home directory. Using `/` on a `Path` joins segments, so `CONFIG_PATH` resolves to `~/.inkly/config.toml` — where Inkly's config lives after installation.

---

## Helper Functions (lines 19–47)

Three small functions, each doing one focused job:

### `_load_config()`
Creates a `TomlParser` pointed at the config file and calls `.load()`. All parsing logic lives in `config.py`; this just kicks it off.

### `_build_user_id()`
```python
return os.environ.get("USER") or os.environ.get("LOGNAME") or "default"
```
Reads the `USER` or `LOGNAME` environment variable to identify who is running the command. On a shared HPC cluster, multiple people use the same machine — this tells Inkly whose conversation history to load.

### `_build_query()`
```python
return " ".join(argv).strip()
```
Joins command-line arguments into one string. If you type `ink how busy is the cluster`, `sys.argv` is `["ink", "how", "busy", "is", "the", "cluster"]`. After slicing off the command name and joining, you get `"how busy is the cluster"`.

---

## `main()` Function (lines 50–102)

The five-step flow:

### Step 1 — Build the query
```python
query = _build_query(sys.argv[1:])
if not query:
    print("Usage: ink <prompt>", file=sys.stderr)
    return 1
```
`sys.argv[1:]` skips the first element (the command name itself). If the result is empty, exit with code `1`.

### Step 2 — Load config
```python
cfg = _load_config()
```
Reads `~/.inkly/config.toml`. Wrapped in `try/except` — if the file is missing or malformed, prints a clean error instead of crashing with a Python traceback.

### Step 3 — Initialize runtime and run
```python
runtime = InklyRuntime(cfg)
user_id = _build_user_id()
response = runtime.handle_query(user_id, query)
```
All real work happens here. Plugins run, the retriever picks what's relevant, the prompt is assembled, and the AI is called. All of that complexity is inside `InklyRuntime`. From this file's perspective, it's just one line.

### Step 4 — Print the response
```python
if not sys.stdout.isatty():
    print(response)
elif response and not response.endswith("\n"):
    print()
```
`sys.stdout.isatty()` checks if output is going to a real interactive terminal or being piped (e.g. into a file or another program).
- **Interactive terminal:** the backend already streamed the response character-by-character while generating it — printing again would duplicate it.
- **Piped:** streaming didn't happen, so print the full response now.

### Step 5 — Exit
```python
return 0
```
`0` = success. `raise SystemExit(main())` at the bottom of the file converts this return value into the actual process exit code.

---

## Key Pattern: Separation of Concerns

`ink_core.py` is intentionally short. It only handles the entry point: receive input → load config → hand off to runtime → return output. Every other piece of complexity lives elsewhere. This pattern repeats throughout the codebase and is worth recognizing as you explore other files.

---

## Next File to Explore

`inkly/core/runtime.py` — contains `InklyRuntime` and the `handle_query` method where all the real pipeline logic (plugin selection, prompt assembly, AI call) lives.

---

## Related Notes

- [[Inkly Codebase Notes Start]] — overview of all key concepts before diving into files
- [[runtime.py]] — the next file in the pipeline, where `handle_query` lives
- [[Testing Inkly]] — how to test the code we explored here
