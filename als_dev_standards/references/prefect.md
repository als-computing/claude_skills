# Prefect Conventions Reference

## Flows vs. Tasks

| | Flow | Task |
|---|---|---|
| **Purpose** | Orchestration logic, high-level steps | A single unit of work |
| **Retries** | Rarely — retry at task level | Yes, configure per task |
| **Logging** | `get_run_logger()` | `get_run_logger()` |
| **State** | Tracks the whole pipeline | Tracks one step |
| **Subflows** | Can call other flows | Cannot call flows |

**Rule of thumb:** if it talks to an external service (Globus, SFAPI, SciCat,
a filesystem), it's a task. The flow only decides what runs and in what order.

```python
# ✅ Flow as orchestrator
@flow(flow_run_name="bl832-ingest-{scan_path}")
def bl832_ingest_flow(scan_path: Path) -> None:
    metadata = extract_metadata(scan_path)       # task
    task_id = submit_transfer(scan_path)         # task
    wait_for_transfer(task_id)                   # task
    ingest_to_scicat(metadata)                   # task

# ❌ Flow doing the work directly
@flow
def bl832_ingest_flow(scan_path: Path) -> None:
    with h5py.File(scan_path) as f:              # should be a task
        metadata = dict(f.attrs)
    globus_client.submit_transfer(...)           # should be a task
```

---

## Run Names

Always set `flow_run_name` and `task_run_name`. Include the file path or primary
identifier so runs are distinguishable in the Prefect UI.

```python
@task(
    name="submit-globus-transfer",
    task_run_name="submit-globus-{src_path}",
)
def submit_globus_transfer(src_path: Path, dst_path: Path) -> str:
    ...

@flow(
    name="bl832-reconstruction",
    flow_run_name="bl832-recon-{scan_path}",
)
def bl832_reconstruction_flow(scan_path: Path) -> None:
    ...
```

The `task_run_name` and `flow_run_name` support `{parameter_name}` interpolation
using the function's parameter names.

---

## Retries and Timeouts

**Always ask the user whether they want retries before adding them.** Do not add
`retries` or `retry_delay_seconds` to a task by default — ask first, then apply
the values the user confirms.

When generating or reviewing Prefect task code, prompt:
> "Do you want to include retries for this task? If so, how many retries and what
> delay between attempts?"

Suggested starting points for common ALS services (use only if user confirms):

| Service | retries | retry_delay_seconds | notes |
|---|---|---|---|
| SFAPI job submission | 3 | 60 | |
| Globus transfer submission | 3 | 30 | polling is a separate task |
| Long-running waits (transfer, job) | 0 | — | use `timeout_seconds` instead |

```python
# Only add retries after confirming with the user
@task(
    retries=3,               # confirmed by user
    retry_delay_seconds=60,  # confirmed by user
    timeout_seconds=300,
)
def submit_sfapi_job(script: Path, site: str) -> str:
    ...
```

Do **not** set retries on flows — retry at the task level so only the failed
step re-runs, not the entire pipeline.

---

## `submit()` vs. Direct Call

| | Direct call | `.submit()` |
|---|---|---|
| Execution | Sequential, blocks | Concurrent via work pool |
| Result | Return value directly | `PrefectFuture` — call `.result()` |
| Use when | Order matters, simple pipelines | Independent tasks can run in parallel |

```python
# Sequential — use direct call
metadata = extract_metadata(scan_path)
task_id = submit_transfer(scan_path, metadata)  # needs metadata first

# Parallel — use submit()
futures = [process_file.submit(f) for f in scan_files]
results = [f.result() for f in futures]
```

---

## Parameters and Variables

- **Flow parameters** (passed at call time): use for per-run values — file paths,
  scan IDs, site names.
- **Prefect Variables** (stored in the server): use for configuration that changes
  infrequently — endpoint IDs, collection names, toggle flags.

```python
from prefect.variables import Variable

@flow
def bl832_ingest_flow(scan_path: Path) -> None:
    # Runtime parameter — passed in
    # Configuration — fetched from Prefect Variables
    collection = Variable.get("bl832_scicat_collection", default="bl832")
    ...
```

---

## Logging Inside Flows and Tasks

Always call `get_run_logger()` at the top of each flow/task body. See
`logging.md` for full conventions.

```python
@task(task_run_name="process-{filepath}")
def process_scan_file(filepath: Path) -> dict[str, Any]:
    logger = get_run_logger()
    logger.info(f"Processing {filepath}")
    ...
```

---

## Subflows

Use subflows to group related tasks that can be independently retried or monitored:

```python
@flow(flow_run_name="transfer-{scan_path}")
def transfer_subflow(scan_path: Path, dst: Path) -> str:
    task_id = submit_globus_transfer(scan_path, dst)
    wait_for_transfer(task_id)
    return task_id

@flow(flow_run_name="bl832-full-{scan_path}")
def bl832_full_pipeline(scan_path: Path) -> None:
    transfer_subflow(scan_path, dst=Path("/global/cfs/..."))
    run_reconstruction(scan_path)
```