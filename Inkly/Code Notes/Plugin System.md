# Plugin System — Detailed Walkthrough

**Files covered:**
- `inkly/plugins/manager.py` — discovers and organizes plugins
- `inkly/plugins/queue_status.py` — a concrete example of one plugin

---

## What Plugins Are

A plugin is a small, self-contained module that collects one specific type of live cluster information and returns it as formatted text. The runtime runs whichever plugins are relevant to the user's question and includes their output in the prompt sent to the AI.

Think of plugins as **reporters**: each one goes out, checks one thing (queue status, node info, job history, etc.), and comes back with a short report.

---

## `manager.py` — The Plugin Manager

### The `Plugin` Dataclass

```python
@dataclass(frozen=True)
class Plugin:
    name: str
    description: str
    category: str
    example_queries: list[str]
    run: Callable[[], str]
```

A **dataclass** is a Python shortcut for creating a class that mainly holds data. `frozen=True` means once a Plugin object is created, its fields cannot be changed — it's immutable.

Every plugin, regardless of which file it came from, gets normalized into this same structure. The `run` field holds the plugin's `run()` function as a callable — meaning you can call it later with `plugin.run()`.

### `PluginManager.discover()`

This is the key method. It dynamically finds all plugins without needing a hard-coded list:

```python
for module_info in pkgutil.iter_modules(package.__path__):
    module_name = module_info.name

    if module_name.startswith("_") or module_name == "manager":
        continue

    module = importlib.import_module(full_name)

    if not hasattr(module, "PLUGIN_META"):
        continue
    if not hasattr(module, "run"):
        continue

    # Build a Plugin object from what was found
```

Step by step:
1. **Iterate** over every Python file in `inkly/plugins/`.
2. **Skip** files starting with `_` (private/internal) and `manager` itself.
3. **Import** each remaining file as a module.
4. **Check** that the module has both `PLUGIN_META` and a `run` function — these are the two required pieces of any plugin.
5. **Build** a normalized `Plugin` object and store it in a dictionary keyed by name.

**Why dynamic discovery?** So that adding a new plugin is as simple as dropping a new `.py` file in the folder. The manager finds it automatically — no registration required.

---

## `queue_status.py` — A Plugin in Practice

This is a complete, real plugin. Reading it shows exactly what a plugin looks like.

### `PLUGIN_META` — The Plugin's Identity Card

```python
PLUGIN_META = {
    "name": "queue_status",
    "description": "Summarizes the current Slurm queue using squeue and sinfo...",
    "category": "queue-status",
    "example_queries": [
        "How busy is the cluster right now?",
        "Are there many pending jobs?",
        "What does the current queue look like?",
        "Is the gpu partition busy?",
    ],
}
```

This dictionary is what the retriever uses to decide whether this plugin is relevant to a question. The `example_queries` list is especially important — these are converted into vectors and compared against the user's actual query to measure similarity.

### `_command_exists()` and `_run_capture()` — Safe Command Execution

```python
def _command_exists(cmd: str) -> bool:
    return shutil.which(cmd) is not None
```

Before running any Slurm command, the plugin checks it's actually available. On a machine without Slurm (like a laptop), `squeue` and `sinfo` don't exist. Without this check, the plugin would crash. With it, it returns a clean fallback message instead.

```python
def _run_capture(cmd: list[str]) -> str | None:
    result = subprocess.run(cmd, stdout=subprocess.PIPE, ...)
    return result.stdout.strip()
```

Runs a shell command and returns its output as a string. Returns `None` if it fails, so the calling code can handle the failure gracefully.

### `_count_jobs_by_state()` — Live Slurm Data

```python
output = _run_capture(["squeue", "-h", "-t", state])
lines = [line for line in output.splitlines() if line.strip()]
return len(lines)
```

Runs `squeue -h -t RUNNING` (or `PENDING`). The `-h` flag suppresses the header row so each output line is exactly one job. Counting lines = counting jobs.

### `_get_partition_lines()` — Partition Summary

```python
output = _run_capture(["sinfo", "-h", "-o", "%P|%D|%C"])
```

Runs `sinfo` with a custom format: `%P` = partition name, `%D` = node count, `%C` = CPU allocation summary. Each row gets parsed and formatted into a readable line like:

```
- gpu: 4 nodes, CPU summary 8/32/0/32
```

### `run()` — The Entry Point

```python
def run() -> str:
    body_lines = []
    running_jobs = _count_jobs_by_state("RUNNING")
    pending_jobs = _count_jobs_by_state("PENDING")
    ...
    return format_plugin_output("Queue Status", body_lines)
```

This is the function the runtime calls. It collects everything (job counts + partition info), assembles the lines, and passes them to `format_plugin_output()` which wraps them in a consistent labeled block.

---

## How a New Plugin Would Look

Based on this pattern, adding a new plugin means creating a file like `inkly/plugins/my_plugin.py` with:

```python
PLUGIN_META = {
    "name": "my_plugin",
    "description": "What this plugin does",
    "category": "some-category",
    "example_queries": ["questions this plugin is relevant to"],
}

def run() -> str:
    # collect some data
    # return it as a formatted string
    return "My plugin output here"
```

The manager finds it automatically on the next run. No other files need to be changed.

---

## Summary of Relationships

```
PluginManager.discover()
    │
    ├── scans inkly/plugins/ for .py files
    ├── checks each for PLUGIN_META + run()
    └── returns dict of Plugin objects
            │
            └── runtime calls plugin.run() on selected ones
                    │
                    └── output goes into the prompt as [PLUGIN CONTEXT]
```

---

## Related Notes

- [[runtime.py]] — calls `plugin.run()` on selected plugins during `handle_query`
- [[retriever.py]] — decides which plugins get selected before they run
- [[Gaussian Scraper Final Thoughts]] — a real plugin built during Work Session 1
- [[Testing Inkly]] — how plugins are tested with fake Slurm output
