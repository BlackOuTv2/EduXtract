# Design Decisions — EduXtract

This document explains *why* certain architectural choices were made — the context that doesn't fit in code comments.

---

## 1. asyncio + aiohttp over threading

**Decision**: The entire download and upload pipeline is built on `asyncio` with `aiohttp` as the HTTP client.

**Why**: Content migrations involve hundreds of network-bound operations (API calls, downloads, S3 uploads). Threads have overhead per operation (~8 KB stack, OS scheduler contention). An async event loop with configurable semaphores (1–20 workers) achieves higher throughput with lower memory pressure.

**Trade-off**: Asyncio on Windows has the ProactorEventLoop which creates noise (WinError 10054 on remote disconnects). Mitigated with a custom exception handler and `_SuppressWinResetFilter` in the logger.

---

## 2. Nuitka over PyInstaller for binary distribution

**Decision**: The build script uses Nuitka (`--standalone --onefile --lto=yes`) instead of PyInstaller.

**Why**:
- Nuitka compiles Python to C, then to native machine code — the output is not Python bytecode wrapped in a zip (as PyInstaller produces)
- LTO (link-time optimization) further obfuscates module boundaries
- This matters for a tool that handles auth tokens and DRM keys — bytecode extraction tools (`unpy2exe`, `pyinstxtractor`) would expose source code in a PyInstaller build
- Build time is longer (5–15 min) but is a one-time cost per release

**Trade-off**: Nuitka requires a C compiler (MinGW on Windows). Added to `requirements.txt` and handled in the build script with a silent `pip install`.

---

## 3. Centralized CSV logger over raw csv.writer

**Decision**: All extractors use a single centralized CSV logger instead of calling `csv.writer` directly.

**Why**: Early prototypes had each extractor maintaining its own CSV file handle, leading to:
- Inconsistent column headers across platforms
- Race conditions when multiple async tasks wrote simultaneously
- No global visibility into what had been logged

`CSVLogger` provides a single write path with:
- A `threading.Lock` per logical file name
- Header validation (rejects rows with unknown columns)
- Logical-to-physical path mapping (the caller uses `"videos"`, not the full filename)

**Trade-off**: Adds an abstraction layer. The cost is negligible; the benefit is consistent output structure across all platforms.

---

## 4. Per-bucket token storage (not a global token store)

**Decision**: ClassPlus tokens are stored one token store per S3 bucket (keyed by bucket name).

**Why**: Each S3 bucket represents a distinct client/academy. Mixing tokens would cause extraction from academy A to accidentally use academy B's token, silently returning wrong content.

Naming tokens by bucket makes the mapping explicit and human-readable. Operators running multiple migrations simultaneously can trivially inspect which token belongs to which academy.

**Trade-off**: Operators managing many clients accumulate many token files. Not a problem in practice — token files are small (< 1 KB) and the naming convention makes them easy to clean up.

---

## 5. Self-destruct on expiry instead of a license server

**Decision**: A dedicated licensing module implements a time-based expiry check using network time, with a self-destruct on expiry. There is no license server or persistent license file.

**Why**: The tool is distributed as a standalone executable to authorized operators. A license server would require:
- Always-on infrastructure
- Network connectivity from the operator's machine to our server
- Server maintenance

A network-verified time check (two independent sources: WorldTimeAPI + Google headers) achieves the same outcome for a per-build license window without infrastructure overhead. The obfuscated fake-error display on expiry is intentional: it makes it non-obvious to unauthorized users that the tool has expired rather than genuinely malfunctioning.

**Trade-off**: System clock manipulation can bypass the check if both network sources fail. Accepted — operators that need to extend their license should contact the developer for a new build.

---

## 6. Mode detection from CSV filename (not a flag)

**Decision**: The processing engine detects the mode (classplus / appx / graphy / normal) by scanning the input CSV filename for keywords.

**Why**: Operators generate CSVs via the extraction options (1–5). The generated filename always contains the platform name (e.g., `<bucket>_m3u8_classplus.csv`). Requiring them to re-specify the platform at upload time is redundant and error-prone.

Filename detection makes the upload step zero-config: the file's name carries the provenance of the data.

**Trade-off**: If an operator manually renames a CSV file, auto-detection breaks. Mitigated by documenting the naming convention and defaulting gracefully to "normal URL mode" on no match.

---

## 7. Error CSV as first-class retry artifact

**Decision**: Failed downloads are logged to a timestamped errors CSV with the exact same schema as the input CSV.

**Why**: A simple error log (just URLs or messages) requires the operator to manually reconstruct the input row to retry. The errors CSV is a drop-in replacement — the operator runs Option 6 with the errors CSV, and everything retries without any manual reconstruction.

This makes partial failures fully self-healing from the operator's perspective.

---

## 8. Async input via asyncio.to_thread

**Decision**: The interactive input helper wraps `input()` with `asyncio.to_thread()`.

**Why**: `input()` is a blocking call. Calling it directly in an async function blocks the entire event loop — any pending async tasks (downloads, uploads, heartbeats) freeze until the user types something.

`asyncio.to_thread(input, prompt)` runs `input()` in a thread pool worker, yielding the event loop while waiting for the user. This keeps background tasks alive during interactive prompts.

**Trade-off**: Adds a layer of indirection. No observable downside in practice.

---

## 9. Single global aiohttp session (singleton)

**Decision**: A singleton HTTP client wraps a single `aiohttp.ClientSession` for all requests.

**Why**: `aiohttp.ClientSession` is designed to be reused. Creating a session per request (or per module) wastes connection pool slots and adds latency. Platform-specific headers are passed per-request, not baked into the session.

**Trade-off**: The singleton must be explicitly closed on exit. Handled in the entry point's `finally` block.

---

## 10. Windows-only (no Linux/macOS support currently)

**Decision**: The project targets Windows exclusively.

**Why**: The external binaries (`m3u8DL.exe`, `mp4decrypt.exe`, `yt-dlp_win.exe`) are Windows executables. The Nuitka build target is Windows. The primary operators are on Windows.

Adding cross-platform support requires either:
- Replacing Windows-specific binaries with cross-platform equivalents (Linux/macOS builds of the same tools exist)
- Wrapping in Docker (eliminates the binary problem entirely)

This is tracked in [ROADMAP.md](./ROADMAP.md) Phase 4 as a high-priority item.
