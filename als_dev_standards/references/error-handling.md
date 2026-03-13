# Error Handling Reference

## The Three Patterns

### 1. Per-item loop: `try / except / continue`

Use when processing a collection where individual failures shouldn't stop the rest.
**Always log before continuing** — never swallow silently.

```python
failed = []
for filepath in scan_files:
    try:
        process_file(filepath)
    except FileNotFoundError as e:
        logger.warning(f"File not found, skipping: {filepath}")
        failed.append(filepath)
        continue
    except Exception as e:
        logger.warning(f"Unexpected error processing {filepath}: {e}")
        failed.append(filepath)
        continue

if failed:
    logger.error("Skipped %d files due to errors: %s", len(failed), failed)
```

### 2. Re-raise with context

Use when you want to add information before letting the exception propagate up
(e.g., which job ID failed, which beamline was involved).

```python
try:
    result = sfapi_client.submit(script, job_id)
except SFApiError as e:
    logger.error(f"SFAPI submission failed for job {job_id} at {compute_site}")
    raise RuntimeError(f"Job submission failed: {job_id} at {compute_site}") from e
```

The `from e` preserves the original traceback. Use it consistently.

### 3. Propagate (do nothing)

Use for truly unexpected failures where there's no meaningful recovery at the
current level. Don't add a `try/except` just to re-log — let it propagate to
Prefect, which will mark the run as FAILED and capture the traceback.

```python
# ✅ Just let it propagate — Prefect handles state and logging
def submit_reconstruction(job_config: JobConfig) -> str:
    return sfapi_client.submit(job_config.script)
```

---

## Warning vs. Error vs. Failure

| Situation | Action | Log level |
|---|---|---|
| Skipping one file in a batch | `continue` | `WARNING` |
| An optional step didn't complete | Continue flow | `WARNING` |
| A required sub-step failed but flow continues | Log and handle | `ERROR` |
| Fatal — flow cannot continue | Raise / propagate | `ERROR` then exception |
| Prefect task failed state | Let Prefect handle | (Prefect logs automatically) |

---

## Never Do This

```python
# ❌ Silent swallow
try:
    process_file(filepath)
except Exception:
    pass

# ❌ Log and re-raise the same exception twice (double logging)
try:
    process_file(filepath)
except Exception as e:
    logger.error(f"Failed: {e}")
    raise  # Prefect will log this too — now it's logged twice

# ❌ Catch-all masking real bugs in development
except Exception as e:
    logger.warning(f"Something went wrong: {e}")
    continue
```

---

## Prefect-Specific: Warnings vs. Flow Failure

Prefect tasks fail their run state when an unhandled exception propagates. Use
this deliberately:

- **Crash the task** (let exception propagate) when the data or state is invalid
  enough that downstream tasks can't safely run.
- **Warn and continue** when the failure is isolated and downstream tasks can
  still produce a meaningful result.

```python
@task
def transfer_file(src: Path, dst: Path) -> None:
    """Transfer a file; crashes the task if transfer cannot start."""
    try:
        task_id = globus_client.submit_transfer(src, dst)
    except GlobusAPIError as e:
        logger.error(f"Globus transfer submission failed: {src} -> {dst}: {e}")
        raise  # Crash the task — downstream tasks depend on this

    # Polling failures are warnings — maybe it'll succeed on retry
    try:
        wait_for_transfer(task_id)
    except TransferTimeoutError:
        logger.warning(f"Transfer {task_id} timed out — may still complete")
```