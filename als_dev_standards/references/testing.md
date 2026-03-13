# Testing Reference

All tests are written with **pytest**. No `unittest`-style test classes or
`TestCase` subclasses in new code.

---

## File and naming conventions

- Test files: `test_<module_name>.py`
- Test functions: `test_<what_is_being_tested>`
- Fixtures: in `conftest.py` at the appropriate directory level
- One logical assertion group per test function
- Use `@pytest.mark.parametrize` for variations instead of loops

---

## General patterns

```python
import pytest
from mymodule import process_file

def test_process_file_returns_metadata(tmp_path):
    """Process file returns a dict of metadata."""
    result = process_file(tmp_path / "scan.h5")
    assert isinstance(result, dict)
    assert "energy" in result

def test_process_file_raises_on_missing_file(tmp_path):
    """Process file raises FileNotFoundError for missing input."""
    with pytest.raises(FileNotFoundError):
        process_file(tmp_path / "nonexistent.h5")

@pytest.mark.parametrize("max_frames,expected", [(1, 1), (5, 5), (0, 0)])
def test_process_file_respects_max_frames(tmp_path, max_frames, expected):
    """Process file returns at most max_frames results."""
    result = process_file(tmp_path / "scan.h5", max_frames=max_frames)
    assert len(result) == expected
```

---

## Mocking external services

Use **`pytest-mock`** (`mocker` fixture) to mock Globus, SFAPI, database calls,
or any I/O boundary. Never use `unittest.mock` directly.

```python
def test_submit_transfer_returns_task_id(mocker):
    """Transfer submission returns a Globus task ID."""
    mock_client = mocker.patch("mymodule.tasks.TransferClient")
    mock_client.return_value.submit_transfer.return_value = mocker.MagicMock(task_id="abc123")
    result = submit_transfer(src="/path/a", dst="/path/b")
    assert result == "abc123"

def test_sfapi_submission_retries_on_failure(mocker):
    """SFAPI submission raises on repeated failure."""
    mocker.patch("mymodule.tasks.sfapi_client.submit", side_effect=RuntimeError("timeout"))
    with pytest.raises(RuntimeError, match="timeout"):
        submit_sfapi_job(script="run.sh", site="perlmutter")
```

Install `pytest-mock` in the `test` extra:

```toml
[project.optional-dependencies]
test = [
    "pytest>=7.0",
    "pytest-mock",
    "pytest-asyncio",
    "prefect[testing]",
]
```

---

## Prefect testing

### Setup fixture

Place this in `conftest.py` at the root of the test suite. `autouse=True,
scope="session"` means it runs once per session and applies to all tests automatically.

```python
from uuid import uuid4

import pytest
from prefect.blocks.system import Secret
from prefect.testing.utilities import prefect_test_harness
from prefect.variables import Variable


@pytest.fixture(autouse=True, scope="session")
def prefect_test_fixture():
    """Set up Prefect test harness with all required secrets and variables."""
    with prefect_test_harness():
        Secret(value=str(uuid4())).save(name="globus-client-id", overwrite=True)
        Secret(value=str(uuid4())).save(name="globus-client-secret", overwrite=True)
        Secret(value=str(uuid4())).save(name="globus-compute-endpoint", overwrite=True)

        Variable.set(
            name="alcf-allocation-root-path",
            value={"alcf-allocation-root-path": "/eagle/test"},
            overwrite=True,
            _sync=True,
        )
        Variable.set(
            name="pruning-config",
            value={"max_wait_seconds": 600},
            overwrite=True,
            _sync=True,
        )
        yield
```

This sets up an ephemeral in-memory Prefect environment — no running server needed.
Add `Secret.save()` and `Variable.set()` calls for any additional secrets or variables
your flows require. Use realistic-shaped values so flow logic that validates inputs
doesn't fail before the code you're actually testing.

For flow/task design conventions, see `prefect.md`.

### Testing a flow

Flows run synchronously inside `prefect_test_harness`:

```python
from mymodule.flows import bl832_ingest_flow

def test_bl832_ingest_flow_completes(tmp_path):
    """Ingest flow completes without error for a valid scan path."""
    result = bl832_ingest_flow(scan_path=tmp_path / "scan.h5")
    assert result is not None
```

### Testing a task in isolation

Call the task's underlying function via `.fn` to bypass Prefect machinery and
test the pure logic directly:

```python
from mymodule.tasks import extract_metadata

def test_extract_metadata_returns_dict(tmp_path):
    """Extract metadata returns a non-empty dict for a valid file."""
    result = extract_metadata.fn(filepath=tmp_path / "scan.h5")
    assert isinstance(result, dict)
    assert len(result) > 0
```