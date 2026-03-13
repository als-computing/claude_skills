# MkDocs Reference

## Setup

Install dependencies in the `dev` extra:

```toml
[project.optional-dependencies]
dev = [
    "mkdocs>=1.5",
    "mkdocs-material>=9.0",
    "mkdocstrings[python]>=0.24",
]
```

---

## `mkdocs.yml` Template

```yaml
site_name: My ALS Project
site_description: Short description of the project.
repo_url: https://github.com/als-computing/my-project

theme:
  name: material
  palette:
    - scheme: default
      primary: indigo

plugins:
  - search
  - mkdocstrings:
      handlers:
        python:
          options:
            docstring_style: google          # matches Args:/Returns:/Raises: format
            show_source: true
            show_root_heading: true

nav:
  - Home: index.md
  - API Reference:
    - Flows: api/flows.md
    - Tasks: api/tasks.md
    - CLI: api/cli.md
```

---

## Auto-generating API Reference

Create a stub Markdown file per module. `mkdocstrings` will pull in the
docstrings automatically:

```markdown
<!-- docs/api/flows.md -->
# Flows

::: my_project.flows.bl832
::: my_project.flows.bl733
```

The `:::` directive tells mkdocstrings to render the module's public API,
using the Sphinx-format docstrings already in the code.

---

## What Gets Documented

| Include | Exclude |
|---|---|
| Public functions and classes | Private functions (`_` prefix) |
| `@flow` and `@task` decorated functions | Internal helpers |
| Pydantic models / dataclasses | Test utilities |
| CLI commands | `conftest.py` |

---

## Running Locally

```bash
mkdocs serve        # live-reload dev server at http://127.0.0.1:8000
mkdocs build        # build static site to site/
```

---

## docs/ Structure

```
docs/
├── index.md              # Project overview and quickstart
├── installation.md       # How to install and configure
├── usage.md              # Common usage patterns and examples
└── api/
    ├── flows.md          # Auto-generated from docstrings
    ├── tasks.md
    └── cli.md
```

`index.md` should contain: what the project does, who it's for, and a minimal
working example. Avoid duplicating content that's already in docstrings.