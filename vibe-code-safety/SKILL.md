---
name: vibe-coding-safety
description: Apply safe software development practices when generating code in a scientific computing environment. Use this skill whenever writing scripts, services, containers, or tools that will be deployed at a research facility or shared computing environment. Covers secrets management, rate limiting, network exposure, containers, logging, and error handling.
---

You are generating code for a scientific computing environment (beamline facility, shared HPC-adjacent systems, shared services). Follow all of the practices below without being asked. If any of these practices conflict with the user's request, flag the issue and suggest a safe alternative.

## Secrets & Credentials

- **Never hardcode secrets.** API keys, passwords, tokens, and credentials must always come from environment variables
- Load environment variables using `python-dotenv` or equivalent; never read them manually with `os.environ.get("PASSWORD", "admin")`
- Always include a `.env.example` with placeholder values and a `.env` in `.gitignore`
- **No default passwords.** Not `admin`, not `password123`, not the instrument or service name
- Generate strong passwords with: `openssl rand -base64 32` or `python -c "import secrets; print(secrets.token_urlsafe(32))"`
- Never bake secrets into Dockerfiles or container images — pass them at runtime via environment variables
- `.env` files on shared systems should be `chmod 600` — readable only by the owner

## Rate Limiting & Service Safety

Shared services are used by many people simultaneously. A script with no rate limiting can take down a service for everyone.

- **Always add rate limiting** when calling any external API or shared service in a loop
- At minimum, add `time.sleep(0.1)` between requests — a tenth of a second is usually enough
- Use **exponential backoff** on errors: wait 1s, then 2s, then 4s. Use the `tenacity` library for this:
  ```python
  from tenacity import retry, wait_exponential, stop_after_attempt
  
  @retry(wait=wait_exponential(multiplier=1, min=1, max=10), stop=stop_after_attempt(5))
  def call_service():
      ...
  ```
- When a service returns a 429 (Too Many Requests), back off — **do not retry immediately**
- Do not retry on 4xx errors other than 429 — a 400 means the request is wrong, not the timing
- Prefer **bulk/batch API calls** over individual requests in a loop. Ask: "does this API support batch requests?"
- If connecting to a database, use a connection pool with an explicit **maximum size**. Never leave it unlimited

## Network Exposure

- **Never bind servers to `0.0.0.0`** unless explicitly deploying for network access. This exposes the service to the entire facility network
- Bind to `127.0.0.1` (localhost) for local development
- **Never run Flask or FastAPI with `debug=True`** on any networked or shared machine. Debug mode exposes an interactive Python console to anyone who can reach the port — this is remote code execution
- Jupyter servers must have a password or token set and should not bind to `0.0.0.0`
- If the application needs to be network-accessible, that decision should go through the computing/infrastructure team

## Docker & Containers

- **Pin base image versions** — use `FROM python:3.11-slim`, not `FROM python:latest`. Unpinned tags change silently
- **Never put data in a container image.** Containers are for code and dependencies only. Data comes in via volume mounts at runtime. Baking in data produces enormous, unmanageable images
- Always include a `.dockerignore` to exclude datasets, `.env` files, local caches, and other junk from the image
- **Never include secrets in the Dockerfile or image layers.** Use environment variables passed at runtime
- Set up **GitHub Actions** to build and push images automatically from the start — don't build manually
- Do not run containers as root unless absolutely required. Add a non-root user in the Dockerfile:
  ```dockerfile
  RUN useradd -m appuser
  USER appuser
  ```
- **Docker Desktop requires a commercial license** for many institutional uses. Prefer: Colima (Mac), Docker Engine directly (Linux), Rancher Desktop or Podman Desktop (Windows/Mac)

## Logging

- **Always add logging** to applications — but audit what gets logged
- Never log secrets, API keys, tokens, passwords, or full config objects that may contain credentials
- Use structured logging with a level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) rather than bare `print()` statements
- When catching exceptions, always log them — never silently discard them

## Error Handling

- **Never use bare `except: pass`** — this swallows errors silently and makes debugging nearly impossible:
  ```python
  # Bad
  try:
      do_important_thing()
  except:
      pass
  
  # Good
  try:
      do_important_thing()
  except Exception as e:
      logger.error(f"Failed to do important thing: {e}")
      raise
  ```
- All network calls must have an explicit **timeout**:
  ```python
  # Bad
  requests.get(url)
  
  # Good
  requests.get(url, timeout=30)
  ```
- **Do not use `pickle`** to serialize/deserialize data from untrusted or external sources — loading a pickle file is arbitrary code execution. Prefer JSON, HDF5, or Zarr

## Destructive Operations

- Any script that **writes, overwrites, or deletes files** must have a `--dry-run` flag that prints what would happen without doing it
- Add an explicit confirmation prompt for irreversible operations
- Never delete or overwrite files as a side effect of a script's main purpose without making it obvious

## Dependencies

- **Pin all dependency versions** in `requirements.txt` — `requests==2.31.0`, not `requests`
- Check what the AI is installing. Typosquatting (`requets` instead of `requests`) is a real attack vector, and some AI-suggested packages may be abandoned or have known CVEs
- Never `pip install` packages at runtime inside a script

## Prompt to Use With Any LLM

Paste this at the start of any coding conversation:

```
I'm writing code in a scientific computing environment. Please follow 
these practices without being asked:

- Store all secrets, API keys, and passwords in environment variables — never hardcoded
- Add rate limiting and time.sleep() for any calls to external APIs or services
- Use exponential backoff on errors with the `tenacity` library
- Prefer bulk/batch API calls over loops of individual requests
- Bind any servers to 127.0.0.1, not 0.0.0.0
- Never use debug=True in Flask or FastAPI
- Add a --dry-run flag to any script that writes or deletes files
- Never log secrets or API keys
- Add explicit timeouts to all network calls
- Catch exceptions explicitly and log them — never use bare except: pass
- Pin all dependency versions in requirements.txt
- Never use pickle to load data from external sources
```