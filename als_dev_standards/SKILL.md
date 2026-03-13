---
name: als_dev_standards
description: "Apply ALS Photon Science Computing software development standards to Python and React/TypeScript code. Use this skill whenever the user asks to write, review, rewrite, or explain code in any ALS project. Triggers include: write a function/flow/task/CLI/component, review this code, does this follow our standards, add docstrings, add type hints, write a test, write a Prefect test, set up pyproject.toml, add logging, add error handling, rewrite this, add JSDoc, write a Vitest test, make a Tiled API call, or any request touching Python code quality, typing, documentation, testing, logging, error handling, CLI design, file paths, project structure, React components, Finch, or Tiled in ALS projects (splash_flows, als_messages, PETIOLE-AI, SYNAPS-AI, tiled-viewer-react, finch, or related beamline projects)."
---

# ALS Software Development Standards

Standards for Python and React/TypeScript code in the ALS Photon Science Computing
team. Apply these whenever generating, reviewing, rewriting, or explaining code.

## How to Use This Skill

| Task | Behavior |
|---|---|
| **Generate** | Apply all relevant standards from the start |
| **Review** | Check each section; list violations with section references |
| **Rewrite** | Preserve logic, fix all violations, summarize changes made |
| **Explain** | Quote the relevant section and give a concrete example |

For detailed reference on any section, read the corresponding file in `references/`.

---

## Python Standards

### 1. Typing

All Python code must be fully statically typed.

- Every function argument must have a type annotation.
- Every function must have a return type annotation, including `-> None`.
- `ClassVar` fields must be annotated with `ClassVar[T]`.
- All dataclass fields must be annotated.
- Use built-in generics: `list[str]`, `dict[str, Any]` — not `List`, `Dict` from `typing`.
- Use `T | None` not `Optional[T]`. Use `X | Y` not `Union[X, Y]`.
- Import `Any` from `typing` only when a more specific type isn't available.

```python
# ✅ Correct
from dataclasses import dataclass
from typing import Any, ClassVar

def process_scan(path: Path, max_frames: int = 100) -> list[dict[str, Any]]: ...

@dataclass
class BeamlineConfig:
    name: str
    energy_kev: float
    enabled: bool = True
    default_timeout: ClassVar[int] = 30

# ❌ Wrong — missing annotations
def process_scan(path, max_frames=100): ...
def run(): pass  # missing -> None
```

---

### 2. Docstrings

All public functions, methods, and classes must have docstrings.

**Format rule:** Use **Google style** for new code. Keep **Sphinx `:param:` style**
when editing existing files — never mix formats within a module.

**Content rules:**
- Lead with a clear one-line summary ending in a period.
- Use type hints in the signature — do not repeat types in the docstring.
- Describe intent, not implementation.
- Keep `Returns:` precise — agents rely on this to chain outputs correctly.
- Be explicit in `Raises:` — helps agents build correct error-handling logic.
- Private (`_` prefix) functions may omit docstrings but benefit from them.

**Google style (new code):**
```python
def load_scan_metadata(filepath: Path, validate: bool = True) -> dict[str, Any]:
    """Load metadata from a scan HDF5 file.

    Reads the top-level metadata group and returns a flat dictionary
    with keys normalized to lowercase with underscores.

    Args:
        filepath: Path to the HDF5 scan file.
        validate: If True, raises ValueError when required keys are missing.

    Returns:
        Dictionary of metadata key-value pairs.

    Raises:
        FileNotFoundError: If filepath does not exist.
        ValueError: If validate is True and required metadata keys are absent.
    """
```

**Sphinx style (existing code only):**
```python
def load_scan_metadata(filepath: Path, validate: bool = True) -> dict[str, Any]:
    """Load metadata from a scan HDF5 file.

    :param filepath: Path to the HDF5 scan file.
    :param validate: If True, raises ValueError when required keys are missing.
    :returns: Dictionary of metadata key-value pairs.
    :raises FileNotFoundError: If filepath does not exist.
    :raises ValueError: If validate is True and required keys are absent.
    """
```

---

### 3. Module Imports

Sort alphabetically within each group. Three groups separated by blank lines:

```python
# 1. Python standard library
import logging
import os
from pathlib import Path
from typing import Any

# 2. Third-party installed packages
import prefect
from globus_sdk import TransferClient
from pydantic import BaseModel

# 3. Local project modules
from myproject.flows import bl832_flow
from myproject.tasks import process_scan
```

Use `isort` (or `flake8-isort`) to enforce ordering automatically.

---

### 4. File Paths

Use `pathlib.Path` for all internal path manipulation.

- **User-facing inputs** (CLI args, API params): accept `Path | str`, normalize to
  `Path` immediately at entry.
- **Internally:** always use `Path` methods (`.parent`, `.stem`, `/` operator).
- **At external API boundaries:** call `str(path)` only where a library requires it.
- Never use `os.path` in new code.

```python
def process_file(filepath: Path | str) -> None:
    filepath = Path(filepath)                        # normalize at entry
    output = filepath.parent / f"{filepath.stem}_processed.h5"
    some_hdf5_lib.open(str(output))                  # str() only at boundary
```

---

### 5. Logging

See `references/logging.md` for full detail. Summary:

- **Inside a `@flow` or `@task`:** use `get_run_logger()` — logs appear in Prefect UI.
- **Everywhere else:** use `logger = logging.getLogger(__name__)` at module level.
- **Never** use `print()` for diagnostic output.
- Prefer verbose logging inside Prefect flows and tasks.

```python
# ✅ Inside Prefect
from prefect import get_run_logger
logger = get_run_logger()
logger.info(f"Processing file: {filepath}")

# ✅ Outside Prefect
import logging
logger = logging.getLogger(__name__)
logger.warning(f"Retrying transfer for task {task_id}")

# ❌ Wrong
print(f"Processing {filepath}")
```

---

### 6. Error Handling

See `references/error-handling.md` for full detail. Summary:

- **Per-item loops:** `try/except/continue` — log a warning before continuing.
- **Fatal errors:** let the exception propagate, or re-raise with context using
  `raise ... from e`.
- Never silently swallow exceptions — always log before `continue` or re-raise.
- Use `logger.warning()` for recoverable issues, `logger.error()` for non-fatal
  failures.

```python
# ✅ Per-item loop
for filepath in scan_files:
    try:
        process_file(filepath)
    except Exception as e:
        logger.warning(f"Skipping {filepath}: {e}")
        continue

# ✅ Re-raise with context
try:
    result = sfapi_client.submit(job)
except SFApiError as e:
    logger.error(f"SFAPI submission failed for job {job_id}")
    raise RuntimeError(f"Job submission failed: {job_id}") from e
```

---

### 7. Prefect Conventions

See `references/prefect.md` for full detail. Summary:

- Flows orchestrate high-level logic; tasks do the actual work.
- Always set `flow_run_name` / `task_run_name` with parameter interpolation.
- Include the file path in run names where applicable.
- **Ask the user** before adding retries — never include `retries` by default.

```python
@task(name="process-scan", task_run_name="process-{filepath}")
def process_scan_file(filepath: Path) -> dict[str, Any]:
    """Process a single scan file and return extracted metadata."""
    ...

@flow(name="bl832-ingest", flow_run_name="bl832-ingest-{scan_path}")
def bl832_ingest_flow(scan_path: Path) -> None:
    """Orchestrate ingestion for a BL832 scan."""
    ...
```

---

### 8. CLI Conventions

- Use **Typer** for all new CLIs; **argparse** only for legacy maintenance.
- `app = typer.Typer()` at module level.
- Entry point: `if __name__ == "__main__": app()`
- Register entry points in `pyproject.toml` under `[project.scripts]`.

```python
app = typer.Typer()

@app.command()
def ingest(
    scan_path: Path = typer.Argument(..., help="Path to the scan directory."),
    dry_run: bool = typer.Option(False, "--dry-run", help="Preview without writing."),
) -> None:
    """Ingest scan data from SCAN_PATH into the data catalog."""
    ...

if __name__ == "__main__":
    app()
```

---

### 9. Unit Testing

See `references/testing.md` for full patterns including Prefect-specific testing. Summary:

- Use **pytest** for all tests — no `unittest`-style classes or `TestCase` subclasses.
- Test files named `test_<module>.py`; test functions named `test_<what_is_tested>`.
- One logical assertion group per test; use `@pytest.mark.parametrize` for variations.
- Use `pytest.raises` for exception testing.
- Mock external services (Globus, SFAPI) with `unittest.mock.patch` or `pytest-mock`.
- For Prefect: use the team's `prefect_test_fixture` in `conftest.py`; test tasks via `.fn`.

```python
import pytest
from mymodule import process_file

def test_process_file_returns_metadata(tmp_path):
    result = process_file(tmp_path / "scan.h5")
    assert isinstance(result, dict)

def test_process_file_raises_on_missing_file(tmp_path):
    with pytest.raises(FileNotFoundError):
        process_file(tmp_path / "nonexistent.h5")

@pytest.mark.parametrize("max_frames,expected", [(1, 1), (5, 5), (0, 0)])
def test_process_file_respects_max_frames(tmp_path, max_frames, expected):
    result = process_file(tmp_path / "scan.h5", max_frames=max_frames)
    assert len(result) == expected
```

---

### 10. Requirements / `pyproject.toml`

Use `pyproject.toml` for all new projects. See `references/pyproject.md` for
a complete template. No `requirements.txt`, `setup.py`, or `setup.cfg` in new code.

---

### 11. Documentation / MkDocs

See `references/mkdocs.md` for full detail.

- MkDocs + `material` theme + `mkdocstrings[python]` for API auto-generation.
- Use **Mermaid** for architecture and flow diagrams.
- Docs in `docs/`; config in `mkdocs.yml`.

---

### 12. GitHub Actions

- Run **pytest** on all PRs and pushes to main.
- Run **flake8** linting on all PRs.
- Build and push **Docker images** on merge to main or version tags.

---

### 13. Secrets & API Keys

- Prefect flows: store secrets as **Prefect Secrets**, retrieve with `Secret.load("name")`.
- Local development: use `.env` files with `python-dotenv`.
- Always commit `.env.example` with placeholder values; add `.env` to `.gitignore`.
- Never hardcode credentials or tokens in source code or logs.

---

## React / TypeScript Standards

See `references/react.md` for full detail.

### Design System
- Consult the **ALS Design System** at https://als-computing.github.io/design-system/ and Storybook at https://als-computing.github.io/design-system/docs/ — authoritative source for colors, icons, components, and patterns.
- Pull colors from **`@als-computing/colors`** (`import { colors } from "@als-computing/colors"`) — no hardcoded hex values.
- Pull icons from **`@als-computing/icons`** (`import { Scan } from "@als-computing/icons"`).
- Use a **token system** for semantic color names (e.g. `color-primary-action`).
- Use **Tailwind CSS** for styling; **shadcn/ui** for base components.

### Finch & Tiled
- Use `@blueskyproject/finch` components and pages — do not create one-off layouts.
- All Tiled server API calls must use the **exported API functions** from
  `@blueskyproject/tiled` — these accept a `config` argument and handle auth
  token injection automatically via the shared axios instance.
- Never write raw `fetch` or `axios` calls to the Tiled server directly — doing
  so bypasses auth and loses the shared config/interceptor setup.
- The library exports functions for node fetching, metadata retrieval, table data,
  and search. See `references/react.md` for the current function list.
- Wrap all Tiled API calls with **TanStack Query** for caching and status.

### Documentation
- Add **JSDoc** to all components, hooks, and utility functions.

### Testing
- Use **Vitest** — every new component needs at least one test per prop.

### Routing & API Calls
- **TanStack Query** for all server state.
- **React Router v7** for client-side routing.

---

## Quick Review Checklist

**Python:**
- [ ] All function args + return types annotated; `ClassVar`/dataclass fields typed
- [ ] No bare `List`, `Dict`, `Optional` — use built-in generics and `X | Y`
- [ ] Google-style docstrings for new code; Sphinx for existing; no type repetition
- [ ] Imports: stdlib → third-party → local, alphabetical within each group
- [ ] `Path` everywhere; no `os.path`; `str(path)` only at external boundaries
- [ ] `get_run_logger()` inside Prefect; `getLogger(__name__)` elsewhere; no `print()`
- [ ] `try/except` always logs before `continue`/re-raise; no silent swallowing
- [ ] Prefect tasks/flows have `flow_run_name`/`task_run_name` with file path
- [ ] Typer for new CLIs; argparse for legacy only
- [ ] Tests use pytest; no unittest classes; `pytest.raises` for exceptions; `parametrize` for variations; Prefect tests use `prefect_test_fixture`
- [ ] `pyproject.toml` with `dev`/`test` extras; no `requirements.txt`

**React / TypeScript:**
- [ ] Colors from `@als-computing/colors`; icons from `@als-computing/icons`; no hardcoded hex values
- [ ] Tailwind for styling; shadcn for base components
- [ ] Finch components reused; no one-off layouts
- [ ] Tiled calls use exported library functions, not raw axios/fetch
- [ ] Tiled calls wrapped in TanStack Query
- [ ] JSDoc on all components, hooks, and utilities
- [ ] Vitest tests present with at least one test per prop
- [ ] React Router v7 for routing; TanStack Query for server state