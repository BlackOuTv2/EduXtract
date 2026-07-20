# Contributing to EduXtract

This is currently a **private/internal tool**. Contributions are by invitation only.

---

## Getting Started

### Prerequisites

- Python 3.10+
- Windows 10/11
- AWS account with S3 access
- Access to at least one supported platform (ClassPlus / AppX / Graphy)

### Development Setup

```bash
# Clone
git clone https://github.com/yourusername/eduxtrack.git
cd eduxtrack

# Virtual environment
python -m venv venv
venv\Scripts\activate

# Install all dependencies
pip install -r requirements.txt

# Configure environment
copy .env.example .env
# Edit .env with your credentials

# Run the application entry point
python <entry-point>
```

---

## Code Standards

### Python Style

- Follow **PEP 8** (4-space indentation, max 120-char lines)
- Use **type hints** on all public functions
- Use **async/await** for all I/O operations — no blocking calls on the event loop
- Prefer `asyncio.to_thread()` for CPU-bound or sync-only operations

### Logging

Use the centralized logger — never `print()` for diagnostic output:

```python
from logger_system import logger

logger.info("Extracting course: %s", course_id)
logger.error("Failed to download: %s — %s", url, error)
```

### Error Handling

- Catch specific exceptions, not bare `except:`
- Log errors with context before re-raising or returning
- Add failed items to the errors CSV — never silently drop them

### CSV Output

Use the centralized CSV logger for all CSV writes. Do not use raw `csv.writer` in extractors:

```python
self.csv_logger.log("videos", {
    "Source Url": url,
    "AWS Url": s3_path,
    "Key": drm_key,
    "Type": "video"
})
```

---

## Commit Messages

Format: `<type>: <short description>`

| Type | When to use |
|---|---|
| `feat` | New feature or platform support |
| `fix` | Bug fix |
| `refactor` | Code restructuring without behavior change |
| `perf` | Performance improvement |
| `docs` | Documentation only |
| `chore` | Build scripts, deps, config |
| `security` | Security-related change |

Examples:
```
feat: add Unacademy platform extractor
fix: handle ClassPlus token refresh race condition
perf: batch S3 dedup checks with 100-worker semaphore
security: load AES key from environment instead of hardcoding
```

---

## Pull Request Process

1. Branch from `main`: `git checkout -b feat/your-feature`
2. Write or update tests if applicable
3. Run `pip-audit -r requirements.txt` — no new high/critical CVEs
4. Ensure the standalone build still compiles
5. Open a PR with a clear title and description of what changed and why
6. Link any related issues

---

## Adding a New Platform

To add support for a new EdTech platform, follow this pattern:

1. **Add a platform extractor** — an async client that enumerates courses and emits the structured CSV
2. **Add a platform handler** — resolves auth headers and delegates to the download engine
3. **Register the platform mode** in the processing engine's dispatch map
4. **Add a menu entry** for the new extraction flow
5. **Document the flow** in [ARCHITECTURE.md](./ARCHITECTURE.md) with a sequence diagram

---

## Reporting Issues

- **Security vulnerabilities**: See [SECURITY.md](./SECURITY.md) — do not open a public issue
- **Bugs**: Open a GitHub issue with: OS, Python version, full error traceback, and steps to reproduce
- **Feature requests**: Open a GitHub issue with: use case, platform, expected behavior

---

## Code of Conduct

See [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md).
