# FAQ — EduXtract

---

## General

**Q: What is EduXtract?**

A: EduXtract is an async Python CLI tool for authorized EdTech operators to migrate course content between platforms. It handles the full pipeline: API extraction → DRM decryption → download → AWS S3 upload.

**Q: Which platforms are supported?**

A: ClassPlus (paid + free), AppX / CMS One, Graphy (formerly Spayee), YouTube, and direct URLs (HTTP/m3u8).

**Q: Is this a piracy tool?**

A: No. EduXtract is designed for B2B content migration by platform operators and content owners. You must have authorization to access and migrate the content. See [SECURITY.md](./SECURITY.md) for the legal scope.

**Q: Does it work on Linux or macOS?**

A: Not currently. The external binaries (`m3u8DL.exe`, `mp4decrypt.exe`) are Windows-only. Linux/macOS support via Docker is planned for Phase 4. See [ROADMAP.md](./ROADMAP.md).

---

## Installation

**Q: Why does the app raise an environment error on startup?**

A: AWS credentials are not set. Copy `.env.example` to `.env` and fill in your `AWS_ACCESS_KEY` and `AWS_SECRET_KEY`. Then run again.

**Q: Where do I get the external binaries (ffmpeg, m3u8DL, mp4decrypt, yt-dlp)?**

A:
- `ffmpeg.exe` + `ffprobe.exe`: [ffmpeg.org/download.html](https://ffmpeg.org/download.html) → Windows builds
- `m3u8DL.exe`: N_m3u8DL-RE releases on GitHub
- `mp4decrypt.exe`: Bento4 SDK releases on GitHub
- `yt-dlp_win.exe`: [github.com/yt-dlp/yt-dlp/releases](https://github.com/yt-dlp/yt-dlp/releases) → `yt-dlp.exe`

Place all binaries in the project root directory.

**Q: What Python version is required?**

A: Python 3.10 or higher. The code uses structural pattern matching syntax and `asyncio.to_thread()` which require 3.10+.

---

## Authentication

**Q: Where do I get a ClassPlus token?**

A: Log into the ClassPlus admin panel in your browser, open DevTools → Network, find any API request, and copy the `Authorization: Bearer <token>` value. Paste it when prompted on first run. The token is stored per bucket and auto-refreshed on subsequent runs.

**Q: My ClassPlus token expired and I don't have a refresh token. What do I do?**

A: Clear the stored token for that bucket. On next run, you'll be prompted to enter a fresh token.

**Q: Where do I get the AppX session-cookie value?**

A: Log into the AppX/CMS One admin panel, open DevTools (F12) → Application → Cookies → find the session cookie. Copy its value and paste it when prompted.

**Q: How long do tokens last?**

A:
- ClassPlus: Typically 7–30 days. EduXtract auto-refreshes via the refresh token when possible.
- AppX session cookie: Typically 24–72 hours depending on platform config.
- Graphy: Session-based — entered on each run, lives in memory only.

---

## Usage

**Q: What's the typical workflow?**

A:
1. Run Option 1/3/5 to extract metadata → generates a CSV file
2. Run Option 6 with that CSV to download and upload everything to S3
3. Run Option 10 to verify all S3 URLs are live
4. On failures: re-run Option 6 with the generated errors CSV

**Q: How many parallel workers should I use?**

A: Start with 3–5 for most platforms. ClassPlus and AppX may rate-limit at higher values. If you see 429 errors in the logs, reduce to 2–3.

**Q: Can I process only videos or only PDFs?**

A: Yes. In Option 6, when prompted for type, enter:
- `v` — videos only
- `p` — PDFs only
- `t` — tests only
- `b` — all (default)

**Q: My download is stuck at a specific item. What do I do?**

A: Press `Ctrl+C` to cancel. The item will appear in the errors CSV. The rest of the batch continues on next retry.

**Q: The tool is showing a weird error like "FFmpeg Error: Encoder initialization failed". Is something broken?**

A: This means your license has expired. Contact the developer for an updated build.

---

## S3 / AWS

**Q: What S3 permissions does the IAM user need?**

A: Minimum permissions:
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:CreateBucket",
    "s3:ListAllMyBuckets",
    "s3:ListBucket",
    "s3:PutObject",
    "s3:GetObject",
    "s3:HeadObject",
    "s3:PutBucketPolicy",
    "s3:PutPublicAccessBlock"
  ],
  "Resource": "*"
}
```

**Q: What region should I use?**

A: `ap-south-1` (Mumbai) is the default and recommended for Indian EdTech platforms. It reduces latency for end-users in India.

**Q: The dedup check takes a long time on large CSVs. Can I skip it?**

A: Yes — when prompted "Check S3 for existing files?", enter `n`. This skips the HEAD check and re-uploads everything (including already-uploaded files). Not recommended unless you're certain the bucket is empty.

---

## Build

**Q: How long does the Nuitka build take?**

A: 5–15 minutes depending on CPU cores and disk speed. The build script uses `--jobs=8` — reduce this on lower-end hardware.

**Q: Do I need to rebuild after every code change?**

A: No. Run the app directly during development. Build the standalone executable only when distributing to operators.

**Q: The build fails with "C compiler not found".**

A: Install MinGW-w64. On Windows, the easiest path is via [winlibs.com](https://winlibs.com/) — download the `x86_64-ucrt` build and add it to your `PATH`.

---

## Errors

**Q: `aiohttp.ClientConnectorError` — connection refused / timeout.**

A: The platform API is down or rate-limiting. Wait a few minutes and retry. If persistent, the platform may have changed its API endpoints.

**Q: `boto3.exceptions.S3UploadFailedError`.**

A: Check your AWS credentials and that the IAM user has `s3:PutObject` permission on the target bucket and region.

**Q: `ValueError: Decryption failed` during AppX processing.**

A: The encrypted link format from the platform may have changed. The AES key/IV constants may need updating. Check AppX API responses for format changes.

**Q: `EnvironmentError: AWS credentials not set`.**

A: See [Configuration](./README.md#-configuration). Copy `.env.example` to `.env` and fill in your credentials.
