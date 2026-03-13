# pyproject.toml Reference

Use `pyproject.toml` for all new Python projects. Do not create `requirements.txt`,
`setup.py`, or `setup.cfg` in new projects.

---

## Complete Template

```toml
[build-system]
requires = [
  "black",
  "flake8",
  "freezegun",
  "pytest",
  "pytest-mock",
  "setuptools"
]

[project]
name = "my-als-project"
version = "0.1.0"
description = "Short description of what this project does."
readme = "README.md"
requires-python = ">=3.10"
license = { text = "BSD-3-Clause" }
authors = [
    { name = "ALS Photon Science Computing", email = "als-computing@lbl.gov" },
]
dependencies = [
    "prefect>=3.0",
    "globus-sdk>=3.0",
    "typer>=0.9",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "flake8",
    "flake8-isort",
    "mypy",
    "pre-commit",
]
test = [
    "pytest>=7.0",
    "pytest-asyncio",
    "prefect[testing]",
]

[project.scripts]
# CLI entry points — maps command name to function
my-cli = "my_project.cli:app"

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.flake8]
max-line-length = 100
extend-ignore = ["E203", "W503"]

[tool.isort]
profile = "black"
line_length = 100

[tool.mypy]
python_version = "3.10"
strict = true
ignore_missing_imports = true
```

---

## Dependency Management Rules

- Pin **exact versions** (`==`) only for deployment manifests, not library packages.
- For libraries, use **minimum version constraints** (`>=`) to allow compatible updates.
- Group related deps together with an inline comment:

```toml
dependencies = [
    # Orchestration
    "prefect>=3.0",
    # Data transfer
    "globus-sdk>=3.0",
    "globus-compute-sdk>=2.0",
    # Data formats
    "h5py>=3.0",
    "zarr>=2.0",
    # CLI
    "typer>=0.9",
]
```

- Dev tools (flake8, mypy, pre-commit) go in `[project.optional-dependencies] dev`.
- Test dependencies go in `[project.optional-dependencies] test`.

Install for development:
```bash
pip install -e ".[dev,test]"
```

---

## `requires-python`

Set to `>=3.10` for new projects (enables `X | Y` union syntax, `match` statements,
and built-in `tomllib`).

---

## Linting with flake8 and isort

Use `flake8` for linting and `isort` for import sorting. Run:

```bash
flake8 .          # lint
isort .           # sort imports
```

Configure both in `pyproject.toml` or a `.flake8` file at the project root.
`flake8-isort` integrates isort checks directly into flake8 so CI only needs one command.

---

## Project Structure

```
my-project/
├── pyproject.toml
├── README.md
├── docs/
│   ├── index.md
│   └── api/
├── src/
│   └── my_project/
│       ├── __init__.py
│       ├── cli.py
│       ├── flows/
│       └── tasks/
└── tests/
    ├── conftest.py
    └── test_*.py
```

Use `src/` layout to avoid import confusion between installed and development code.