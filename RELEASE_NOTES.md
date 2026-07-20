# Release Notes — EduXtract

---

## v1.1.0 (2026-07-14)

### What's New

**New CLI options:**
- **Option 4** — AppX Extraction (Requests) — synchronous extractor with `latest_mode` flag for processing newest content first; useful for incremental updates
- **Option 8** — CSV Column Remover — strip unwanted columns from any CSV file interactively
- **Option 9** — CSV URL Cleaner — fix over-encoded URLs in CSV files before processing
- **Option 10** — S3 URL Status Checker — verify up to 100 URLs concurrently, outputs `_checked.csv` and `_uploaded_only.csv`
- **Option 11** — Make Bucket Public — enables anonymous read access on a bucket with idempotency (safe to run multiple times)

**Token improvements:**
- ClassPlus tokens now support dual-format storage: legacy string (access token only) and new `{access_token, refresh_token}` JSON object
- Expired tokens are silently refreshed via the refresh token; operator prompt only appears on full auth failure
- Pre-upload token validation: ClassPlus token is validated before starting a batch CSV run

**Upload intelligence:**
- S3 deduplication check before batch uploads: existing files are skipped (saves time and bandwidth on partial retries)
- Prompt before batch upload: operator can choose to skip the dedup check for fresh buckets

### Breaking Changes

None. All existing CSV formats and token files are backwards compatible.

### Known Issues

- `--latest_mode` on AppX Requests extractor may miss content added more than 30 days ago if the API paginates beyond page 10. Full extraction (non-latest mode) is recommended for initial migrations.

### Upgrade from v1.0.0

No migration steps required. Existing token files and CSV files are fully compatible.

---

## v1.0.0 (2026-06-28)

### Initial Release

First production-ready release with:
- ClassPlus paid + free content extraction
- AppX v2 async extractor
- Graphy extraction
- AWS S3 upload with batch CSV processing
- DRM decryption (Widevine + AES-CBC)
- Rotating logger with colored console
- Nuitka standalone `.exe` build
- Error CSV for retry
- Per-bucket token management
- Time-based license expiry

See [CHANGELOG.md](./CHANGELOG.md) for the full change history.
