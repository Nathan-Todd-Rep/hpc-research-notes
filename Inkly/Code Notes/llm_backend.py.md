# `llm/backend.py`

**File location:** `inkly/llm/backend.py`  
**Role:** The layer that actually talks to Ollama. Sends the assembled prompt and streams the response back.

---

## What This File Does

Once `runtime.py` has assembled the full prompt, it calls `self.backend.generate(prompt)`. This file is responsible for everything that happens at that point: picking the right way to reach Ollama, sending the prompt, and returning the AI's answer as a string.

---

## `_clean_terminal_output()` — Helper Function

```python
def _clean_terminal_output(text: str) -> str:
```

When Ollama runs in a non-interactive context (e.g. piped output), it can pollute its output with:
- **ANSI escape sequences** — special character codes that terminals use to move the cursor, change colors, etc. They look like `\x1b[?25l`.
- **Spinner frames** — characters like `⠋ ⠙ ⠹` that form an animated spinner in a terminal.
- **Carriage returns (`\r`)** — used for overwriting lines in a terminal.

This function strips all of that noise so only the real text remains. It's only needed in non-interactive mode; in interactive mode Ollama streams cleanly character-by-character.

---

## `LLMBackend` Class

### `__init__()`
```python
self.config = config
self.ollama_tunnel = None
self._stdout_lock = threading.Lock()
```
- `ollama_tunnel` starts as `None` — the SSH tunnel manager is only created if the `ssh_tunnel` mode is actually used.
- `_stdout_lock` is a threading lock. When streaming output character-by-character, the lock prevents two threads from writing to the terminal at the same time.

---

## The Spinner

```python
def _spinner_worker(self, stop_event, started_output_event):
    frames = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"]
```

While waiting for the AI to start generating, a spinner (`"Inkly is thinking ⠋"`) is shown in the terminal. This runs on a **separate thread** so it can animate while the main thread waits for output. As soon as the first character of the AI's response arrives, the spinner thread is stopped and the line is cleared with spaces.

---

## Two Ways to Run a Command

The backend has two internal methods for running Ollama as a subprocess:

| Method | When Used | How It Works |
|---|---|---|
| `_stream_command()` | Interactive terminal (`isatty() = True`) | Reads output one character at a time and prints each character immediately |
| `_run_command_capture()` | Non-interactive / piped output | Waits for the command to finish, then returns the full output at once |

Both methods:
1. Launch Ollama as a **subprocess** using `subprocess.Popen` or `subprocess.run`
2. Send the prompt via **stdin** (standard input) rather than as a command-line argument — this avoids hitting argument length limits on very long prompts
3. Check the return code and raise a clean `RuntimeError` if something went wrong
4. Raise an error if the output is empty

A **subprocess** is a separate program launched by Python. Inkly launches `ollama run <model>` as a subprocess the same way you would type it in a terminal, but it captures the input and output programmatically.

---

## `generate()` — The Public Entry Point

```python
def generate(self, prompt: str) -> str:
    prompt = prompt[:self.max_prompt_length()]
    return self._generate_ollama(prompt)
```

Trims the prompt to the maximum configured length, then routes to `_generate_ollama()`.

---

## `_generate_ollama()` — Transport Mode Routing

```python
mode = getattr(ollama_cfg, "mode", "cli_run")

if mode == "admin_command":
    return self._generate_ollama_admin_command(prompt)
if mode == "direct_host":
    return self._generate_ollama_cli_run(prompt, use_direct_host=True)
if mode == "ssh_tunnel":
    return self._generate_ollama_cli_run(prompt, use_tunnel=True)
return self._generate_ollama_cli_run(prompt)
```

Reads the `mode` field from config and routes to the right transport. The four modes and what they do:

| Mode | What Happens |
|---|---|
| `admin_command` | Runs an admin-provided executable path directly (e.g. `/opt/ollama/ollama`) |
| `direct_host` | Sets `OLLAMA_HOST` environment variable and runs `ollama run <model>` against that remote server |
| `ssh_tunnel` | Opens an SSH tunnel first, then runs `ollama run <model>` through it |
| `cli_run` | Runs `ollama run <model>` locally (simplest mode, used for development) |

---

## How Streaming Works

In interactive mode, `_stream_command()` reads the subprocess output **one character at a time**:

```python
while True:
    ch = process.stdout.read(1)
    if ch == "":
        break
    chunks.append(ch)
    sys.stdout.write(ch)
    sys.stdout.flush()
```

- `read(1)` reads exactly one character.
- Each character is immediately written to the terminal with `sys.stdout.write(ch)` and `flush()` — this is what creates the live character-by-character streaming effect.
- All characters are also collected in `chunks` so the full response can be returned as a string to the runtime.

---

## Key Design Decision

The backend always communicates with Ollama by launching it as a subprocess (`ollama run <model>`), sending the prompt on stdin, and reading the output. This is different from calling an HTTP API. The result is the same — a text response — but the mechanism is command-line rather than network-based. This keeps Inkly compatible with shared HPC environments where HTTP access to internal services may be restricted.

---

## Related Notes

- [[runtime.py]] — calls `backend.generate()` as the final step of `handle_query`
- [[Inkly Codebase Notes Start]] — overview of how Ollama fits into the system
- [[Testing Inkly]] — how the backend is tested with fake subprocess calls
