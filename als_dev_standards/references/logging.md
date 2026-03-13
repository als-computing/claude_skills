# Logging Reference

## The Core Rule: Prefect vs. Standard Logger

| Context | Logger to use |
|---|---|
| Inside a `@flow` or `@task` function | `from prefect import get_run_logger` |
| Module-level / outside Prefect | `logging.getLogger(__name__)` |
| Scripts, CLIs, utilities | `logging.getLogger(__name__)` |
| **Never** | `print()` |

Prefect's `get_run_logger()` routes log messages to the Prefect UI alongside the
flow/task run. Standard `logging` loggers do not. Using the wrong one means logs
either go missing from the UI or (worse) cause errors when called outside a run context.

---

## Standard Logger (outside Prefect)

Always declare the logger at **module level**, not inside functions:

```python
import logging

logger = logging.getLogger(__name__)  # __name__ = module path, e.g. "bl832.ingest"

def process_scan(filepath: Path) -> dict[str, Any]:
    logger.info(f"Starting scan processing: {filepath}")
    ...
    logger.debug(f"Parsed {frame_count} frames from {filepath}")
```

`__name__` automatically gives the logger a meaningful hierarchical name matching
the module's import path, which makes filtering in log output easy.

---

## Prefect Logger (inside flows and tasks)

```python
from prefect import flow, task, get_run_logger

@task(task_run_name="process-{filepath}")
def process_scan_file(filepath: Path) -> dict[str, Any]:
    logger = get_run_logger()
    logger.info(f"Processing {filepath}")
    ...

@flow(flow_run_name="bl832-ingest-{scan_path}")
def bl832_ingest_flow(scan_path: Path) -> None:
    logger = get_run_logger()
    logger.info(f"Flow started for {scan_path}")
    process_scan_file(scan_path)
```

Call `get_run_logger()` at the **top of each function body**, not at module level —
it must be called within an active run context.

---

## Log Levels

| Level | When to use |
|---|---|
| `DEBUG` | Fine-grained details useful only when diagnosing a problem |
| `INFO` | Normal operational events — file started, transfer completed |
| `WARNING` | Something unexpected but recoverable — retrying, skipping a file |
| `ERROR` | A failure that doesn't stop the overall process |
| `CRITICAL` | Reserved for situations that will halt execution entirely |

When in doubt: `INFO` for "this happened", `WARNING` for "this was unexpected but OK",
`ERROR` for "this failed and something was lost".

## What to Log

**At flow/task entry and exit:**
```python
logger.info(f"Starting ingest for scan {scan_path}")
# ... work ...
logger.info(f"Ingest complete for {scan_path}: {count} files processed")
```

**On transfer operations (Globus, SFAPI):**
```python
logger.info(f"Submitting Globus transfer: {src} -> {dst}")
logger.info(f"Transfer task ID: {task_id}")
logger.info(f"Transfer complete: {task_id}")
```

**On skipped/failed items:**
```python
logger.warning(f"Skipping {filepath}: unsupported file format {suffix}")
logger.error(f"Failed to process {filepath}: {exc}")
```

---

## What NOT to Log

- Secrets, tokens, or credentials (even in DEBUG)
- Full file contents or large data structures
- Repeated identical messages in tight loops (use a counter and log once)