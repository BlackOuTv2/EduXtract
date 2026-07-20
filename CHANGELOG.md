# Changelog — EduXtract

All notable changes to this project are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versions follow [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

### Planned
- Cross-platform support (Linux/macOS via Docker)
- Test suite (pytest + async fixtures)
- Resume interrupted downloads via checkpoint file

---

## [1.1.0] — 2026-07-14

### Added
- **AppX Requests Extractor** — synchronous fallback extractor with latest-content-first mode
- **CSV Column Remover** (Option 8) — interactive tool to strip columns from any CSV
- **CSV URL Cleaner** (Option 9) — normalizes aggressive percent-encoded URLs in CSV files
- **S3 URL Status Checker** (Option 10) — concurrent HEAD checks for up to 100 S3 URLs in parallel; outputs a full report plus an uploaded-only CSV
- **Make Bucket Public** (Option 11) — enables anonymous read access on an S3 bucket with idempotency check
- **AppX Token Management** — per-bucket API token files supported

### Changed
- **Token management** now supports both legacy string format and new `{access_token, refresh_token}` JSON dict format — backwards compatible
- **ClassPlus token auto-refresh** — expired tokens are silently refreshed via the refresh token; operator is only prompted on full auth failure
- **Menu** reorganized into three logical sections: Extraction, Processing, System
- **Concurrent S3 dedup check** — `check_s3_choice` prompt added before batch uploads; skips already-uploaded files

### Fixed
- Nuitka build: `asyncio.WindowsProactorEventLoopPolicy` explicitly set to fix subprocess spawning in frozen `.exe`
- Silenced harmless Windows asyncio Proactor errors (`WinError 10054`, `_call_connection_lost`) in both logger and event loop handler
- `_quiet_excepthook` prevents noisy `KeyboardInterrupt` / `CancelledError` tracebacks on exit

---

## [1.0.0] — 2026-06-28

### Added
- **ClassPlus Extractor** — full paid course extraction (videos, PDFs, tests) with batch processing and JWT token refresh
- **ClassPlus Free Content Extractor** — public content extraction with content-type selection
- **AppX Extractor** (async, v2) — course extraction via CMS API with session-cookie auth
- **Graphy Extractor** — course catalog + DRM-protected video + PDF extraction
- **AWS Upload Manager** — batch CSV processing with configurable parallel workers (1–20)
- **CSV Generator** — normal URL CSV generation from course data
- **Centralized CSV logger** — thread-safe, header-validated CSV writes replacing all raw `csv.writer` usage
- **Async singleton HTTP client** — single shared `aiohttp.ClientSession` across all modules
- **AES-CBC decryption** — AppX encrypted link decryption + JWT decode utility
- **Token manager** — per-bucket persistence for ClassPlus tokens
- **Rotating logger** — 5 MB × 5 backups, color-coded console output, Windows asyncio noise suppression
- **Time-based license expiry** — network-verified (WorldTimeAPI + Google fallback) with obfuscated error display
- **Nuitka build** — standalone executable with LTO and all binaries bundled
- **Error retry CSV** — failed downloads auto-logged with identical structure to input CSV for one-click replay

### Refactored (from pre-1.0 prototypes)
- Removed dead extractor and token-download modules (unused code paths)
- Reduced the AppX download module from 43 KB to ~3 KB by removing dead multiprocessing code
- Fixed a recursive bug in the file-cleanup helper
- Removed unused token helper functions
- Graphy session reuse: replaced per-run session creation with a class-level `aiohttp.ClientSession`
- Graphy PDF AES decryption offloaded to `asyncio.to_thread()` to prevent event loop blocking

---

## [0.x] — Internal Prototypes (2026-01-19 to 2026-06-27)

- Early per-platform scripts without shared infrastructure
- Manual CSV writing per extractor
- Synchronous-only download flows
- No centralized logging or error tracking
