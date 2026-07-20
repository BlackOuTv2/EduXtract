# 📖 Usage Guide — EduXtract

> EduXtract is a closed-source tool distributed as a standalone executable to authorized operators. This guide describes operation; it does not document internal implementation.

## Starting the App

Launch the application. You'll see the interactive menu:

```
┌── Extraction & Administrative Tools ──┐
│  1. ClassPlus Extraction              │
│  2. ClassPlus Free Content            │
│  3. AppX Extraction                   │
│  4. AppX Extraction New               │
│  5. Graphy Extraction                 │
├── Processing & Data Output Tools    ──┤
│  6. AWS Upload / Process CSV          │
│  7. CSV Generator (Normal URLs)       │
│  8. CSV Remove Column                 │
│  9. CSV URL Cleaner                   │
│ 10. Check S3 URLs Upload Status       │
├── System Operation                  ──┤
│ 11. Make Bucket Public                │
│  0. Exit                              │
└───────────────────────────────────────┘
```

---

## Menu Options

### 1️⃣ ClassPlus Extraction

Extracts all paid course content (videos, PDFs, tests) from a ClassPlus organization.

**Steps:**
1. Select option `1`
2. Choose or create an S3 bucket (this becomes the org identifier)
3. The stored ClassPlus token for that bucket loads automatically; if none exists, you'll be prompted to enter one
4. Enter Course IDs (comma-separated) or press Enter for all courses
5. Set batch size (default: 10 courses processed in parallel)

**Output:** Structured CSVs (tests, PDFs, videos) plus staged downloads.

---

### 2️⃣ ClassPlus Free Content

Extracts free/public content from ClassPlus organizations.

**Steps:**
1. Select option `2`
2. Choose S3 bucket
3. Token loads/entered same as option 1
4. Choose content type: PDF · Video · Test · Test Portal · All (default)

---

### 3️⃣ / 4️⃣ AppX Extraction

Extracts course content from AppX-based platforms (TeachX / CMS One). Option 4 is a requests-based variant with a latest-content-first mode.

**Steps:**
1. Select option `3` or `4`
2. Enter the session cookie value (from your browser)
3. Enter an output base name (e.g., `my-academy`)
4. Extraction runs in batches

**How to get the session cookie:**
1. Log into the AppX admin panel in your browser
2. Open DevTools → Application → Cookies
3. Copy the session cookie value

---

### 5️⃣ Graphy Extraction

Extracts course content from Graphy (formerly Spayee) platforms.

**Steps:**
1. Select option `5`
2. Choose S3 bucket
3. Enter Graphy email and password
4. Optionally enter a specific Course ID, or leave blank for all courses

---

### 6️⃣ AWS Upload / Process CSV

The main batch processing engine. Takes a CSV file and downloads + uploads all content to S3.

**Steps:**
1. Select option `6`
2. Choose S3 bucket
3. Enter CSV filename (without `.csv` extension)
4. Platform mode is auto-detected from the filename:
   - Contains `appx` → AppX mode (prompts for session cookie and API token)
   - Contains `classplus` → ClassPlus mode
   - Contains `graphy` → Graphy mode (prompts for email/password)
   - Otherwise → Normal URL mode
5. Choose upload type: `v` videos · `p` PDFs · `t` tests · `b` both (default)
6. Set parallel workers (1–20, default: 5)

**Expected CSV Format:**

| Column | Description |
|--------|-------------|
| `Source Url` | Original download URL (m3u8, YouTube, direct) |
| `AWS Url` | Target S3 path for upload |
| `Key` | *(Optional)* Decryption key for encrypted content |
| `Type` | *(Optional)* Content type: `video`, `pdf`, `test` |

---

### 7️⃣–1️⃣1️⃣ Processing & System Tools

| Option | Purpose |
|---|---|
| 7. CSV Generator | Generate a clean CSV of direct download URLs from course data |
| 8. CSV Remove Column | Strip selected columns from a CSV interactively |
| 9. CSV URL Cleaner | Normalize percent-encoded URLs in a CSV |
| 10. Check S3 URLs | Verify which S3 URLs are live (concurrent HEAD checks) |
| 11. Make Bucket Public | Grant anonymous read access to a bucket (with confirmation) |

---

## ⚙️ Configuration

### AWS Credentials

All secrets are supplied via environment variables (a `.env` template is provided):

```env
AWS_ACCESS_KEY=your_access_key
AWS_SECRET_KEY=your_secret_key
AWS_REGION=ap-south-1
```

### Token Storage

| Platform | Storage |
|------|---------|
| ClassPlus | Per-bucket local token store (gitignored) |
| AppX | Local session-cookie file (gitignored) |
| Graphy | In-memory only |

---

## 📋 Logs & Error Handling

| Artifact | Description |
|------|-------------|
| Application log | Full event log (rotates automatically) |
| Errors CSV | Failed items, same schema as the input CSV |

### Re-processing Failed Downloads

1. The errors CSV shares the input CSV schema
2. Run **Option 6** with the errors CSV to retry failed items — no manual reconstruction needed

---

## 💡 Tips

- **First Run?** Start with Option 1 or 3 to extract content → generates CSV → then use Option 6 to upload
- **Large Batches?** Use 3–5 parallel workers to avoid rate limiting
- **Token Expired?** Clear the stored token for that bucket and re-enter on next run
- **Download Stuck?** Press `Ctrl+C` to cancel the current operation and return to the menu
