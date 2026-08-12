# Testing Inkly

## What Is a Test?

A test is a small program that runs your real code and checks that it behaves correctly. Instead of manually running `ink <question>` and reading the output every time you make a change, you write a test once and the computer checks it for you automatically.

If you later break something, the test fails immediately and tells you exactly what went wrong — before it ever reaches real users.

---

## The Tool: pytest

Inkly uses **pytest** — the most common Python testing library.

The rules are simple:
- Test files live in the `tests/` folder and are named `test_*.py`
- Individual tests are functions that start with `test_`
- pytest finds and runs them all automatically

```python
def test_build_query_section():
    runtime = InklyRuntime(make_config())
    section = runtime._build_query_section("How do I use the gpu partition?")

    assert "=== USER QUERY ===" in section
    assert "How do I use the gpu partition?" in section
```

---

## The `assert` Statement

An `assert` is the core of every test. It means: *"I claim this is true — if it's not, fail loudly."*

```python
assert response == "final answer"           # fails if response is anything else
assert "Running jobs: 3" in output          # fails if that text isn't in output
assert "=== CONVERSATION HISTORY ===" not in prompt  # fails if that text IS present
```

If an assert fails, pytest stops that test, shows you exactly which line failed and what the actual value was, and marks it as FAILED.

---

## The Core Problem: Fake Dependencies

Inkly's real code depends on things that don't exist on a laptop:

- `squeue` and `sinfo` — Slurm commands that only exist on an HPC cluster
- Ollama — an AI server that may not be running
- A real conversation history database

If tests called all of those for real, they would fail on any laptop, be slow, and be hard to control.

The solution is **fakes** — simplified stand-in objects that behave like the real thing but are fully under your control.

### Example: FakeBackend

```python
class FakeBackend:
    def __init__(self, response="mock response"):
        self.prompts = []           # record every prompt sent to us

    def generate(self, prompt):
        self.prompts.append(prompt) # store it so tests can inspect it later
        return self.response        # return whatever we told it to return
```

This is not a real AI backend — it just stores whatever prompt it receives and returns a preset answer. But it has the same `generate()` method as the real `LLMBackend`, so the runtime can't tell the difference.

The same pattern is used for `FakePlugin`, `FakePluginManager`, and `FakeConversationManager` in `tests/test_runtime.py`.

---

## `monkeypatch` — Swapping Real Things for Fakes

`monkeypatch` is a pytest tool that temporarily replaces something real with a fake — just for the duration of one test. When the test ends, everything is restored automatically.

### Replacing a whole class (test_runtime.py)

```python
monkeypatch.setattr(
    "inkly.core.runtime.LLMBackend",   # target: the real class in that module
    lambda cfg: backend,               # replacement: our FakeBackend instead
)
```

Now when the runtime does `LLMBackend(config)`, it gets our fake instead of the real one.

### Replacing a single function (test_queue_status_plugin.py)

Instead of replacing a whole class, you can replace just one function:

```python
def fake_run_capture(cmd):
    if cmd == ["squeue", "-h", "-t", "RUNNING"]:
        return "job1\njob2\njob3"   # pretend 3 jobs are running
    if cmd == ["squeue", "-h", "-t", "PENDING"]:
        return "job4\njob5"         # pretend 2 jobs are pending
    return None

monkeypatch.setattr(queue_status, "_run_capture", fake_run_capture)
```

Now when `queue_status.run()` internally calls `_run_capture(["squeue", ...])`, it gets fake data back. The test can verify `"Running jobs: 3"` appears in the output — without ever touching a real cluster.

---

## The Three-Step Test Pattern

Every test in this project follows the same structure:

```
1. ISOLATE  — replace real dependencies with fakes
2. CALL     — run the real function you want to test
3. ASSERT   — check the output is what you expected
```

Example from `test_queue_status_plugin.py`:

```python
def test_run_reports_queue_counts_and_partitions(monkeypatch):
    # 1. ISOLATE — fake out the Slurm commands
    monkeypatch.setattr(queue_status, "_command_exists", lambda cmd: True)
    monkeypatch.setattr(queue_status, "_run_capture", fake_run_capture)

    # 2. CALL — run the real plugin
    output = queue_status.run()

    # 3. ASSERT — check the output
    assert "Running jobs: 3" in output
    assert "Pending jobs: 2" in output
    assert "- general: 10 nodes" in output
```

---

## Setting Up the Test Environment

Before running tests for the first time on a new machine, you need to install pytest. Python on Windows uses the `py` launcher.

### Step 1 — Bootstrap pip (if missing)

pip is the Python package installer. If it isn't available:

```bash
py -m ensurepip --upgrade
```

### Step 2 — Install pytest

```bash
py -m pip install pytest
```

### Step 3 — Run from the project folder

All test commands must be run from inside the `hpc-ink-setup` directory:

```bash
cd "C:\Users\Nathan\Documents\GitHub\hpc-ink-setup"
```

---

## Running the Tests

```bash
# Run all tests (short summary)
py -m pytest tests/

# Run all tests with each test name shown
py -m pytest tests/ -v

# Run just one test file
py -m pytest tests/test_queue_status_plugin.py -v

# Run one specific test by name
py -m pytest tests/test_runtime.py::test_build_query_section -v
```

The `-v` flag means "verbose" — shows every individual test name as it runs.

---

## Reading Test Results

```
tests/test_runtime.py::test_build_query_section PASSED        [ 96%]
tests/test_runtime.py::test_assemble_prompt_omits_empty_sections PASSED [ 97%]

2 failed, 93 passed in 4.98s
```

- Each line shows the file, test name, and result.
- The final line gives the overall count and total time.
- If a test fails, pytest prints the exact line that failed and what the actual value was.

### What Failure Output Looks Like

```
AssertionError: assert 'recent-job' in []
tests\test_jobs_db_flow.py:89: AssertionError
```

This tells you: the file, the line number, and what the mismatch was. You don't have to guess — pytest shows you exactly what went wrong.

---

## The Two Failing Tests (a Real Example)

When we ran the suite, 2 tests failed in `test_jobs_db_flow.py`:

```
FAILED tests/test_jobs_db_flow.py::test_cleanup_old_jobs_removes_rows_before_cutoff
FAILED tests/test_jobs_db_flow.py::test_ingest_jobs_to_db_window_stabilizes_on_repeat
```

Both tests used hardcoded dates from a few months ago (jobs from December 2025 and April 2026). The cleanup logic deletes jobs older than 30 days — and since the current date is May 2026, those jobs now fall outside the window and get deleted too.

**These are not bugs in the code — they are bugs in the tests themselves.** The tests were written when those dates were recent, but time passing made them stale.

This is a known hazard called **date-dependent tests**. The fix is to either:
- Use relative dates (e.g. "30 days ago" instead of a fixed date)
- Fake out the clock so the test controls what "now" means

This is a good example of why test quality matters just as much as code quality.

---

## Where the Test Files Live

| Test File | What It Tests |
|---|---|
| `test_runtime.py` | The full `handle_query` pipeline and prompt assembly |
| `test_queue_status_plugin.py` | The queue status plugin with fake Slurm output |
| `test_llm_backend.py` | The Ollama backend with fake subprocess calls |
| `test_conversation.py` | Conversation history storage and context building |
| `test_jobs.py` | Job record parsing and classification logic |
| `test_retrieval.py` | Plugin retrieval and vector search |
| `test_jobs_db_flow.py` | Database insert, update, and cleanup logic |

---

## Key Takeaway

The entire test suite runs in under 5 seconds on a laptop with no cluster, no AI, and no database. That's the power of fakes and monkeypatching — you isolate the piece you're testing, control all its inputs, and verify its outputs. If all 93 tests pass, you can be confident the core logic is working correctly before you ever deploy to a real cluster.

---

## Related Notes

- [[runtime.py]] — the pipeline whose tests are covered in `test_runtime.py`
- [[Plugin System]] — the plugin system tested in `test_queue_status_plugin.py`
- [[Gaussian Scraper Final Thoughts]] — the scraper whose tests were written in Work Session 1
- [[Test Documentation 1]] — the actual test code from Work Session 1
- [[Work Session 1]] — the session where scraper tests were written
