# Troubleshooting — EduXtract

---

## Startup Issues

### `EnvironmentError: AWS credentials not set`

**Cause**: The app requires `AWS_ACCESS_KEY` and `AWS_SECRET_KEY` from the environment.

**Fix**:
```bash
copy .env.example .env
# Open .env and fill in your AWS keys, then relaunch
```

---

### `ModuleNotFoundError: No module named 'aiohttp'`

**Cause**: Dependencies not installed.

**Fix**:
```bash
venv\Scripts\activate
pip install -r requirements.txt
```

---

### `ImportError: DLL load failed while importing _cffi_backend`

**Cause**: `cffi` / `pycryptodome` compiled for a different Python version.

**Fix**:
```bash
pip install --force-reinstall cffi pycryptodome cryptography
```

---

### Random error on startup like `"AWS S3 Connection: SignatureDoesNotMatch violation"`

**Cause**: Your license period has expired. The tool displays contextually realistic fake errors on expiry.

**Fix**: Contact the developer for an updated build.

---

## Authentication Issues

### ClassPlus: `Token validation failed` / `401 Unauthorized`

**Diagnosis**:
1. Check the application log for the exact API response
2. Verify the token hasn't expired (ClassPlus tokens typically last 7–30 days)

**Fix**: Clear the stored token for that bucket and relaunch — you'll be prompted for a fresh token.

---

### AppX: session cookie rejected / `403`

**Cause**: The session cookie has expired (typically 24–72 hours).

**Fix**:
1. Re-login to the AppX admin panel in Chrome
2. DevTools → Application → Cookies → copy the session cookie
3. Paste the new value when prompted on next run

---

### Graphy: `Login failed / invalid credentials`

**Cause**: Wrong email/password, or the Graphy organization has changed auth settings.

**Fix**: Verify credentials by logging in at the Graphy web panel. Graphy does not persist credentials — re-enter on each run.

---

## Download Issues

### Downloads getting stuck / hanging

**Cause**: The HLS downloader subprocess isn't terminating, or the stream URL has expired.

**Fix**:
1. Press `Ctrl+C` to cancel the current operation
2. Check the errors CSV — the stuck item will be logged
3. Re-run Option 6 with the errors CSV
4. If a specific URL always hangs, it may have expired — re-run extraction to get fresh URLs

---

### `ValueError: Decryption failed` for AppX content

**Cause**: AppX may have updated their encryption parameters or the link format changed.

**Diagnosis**: Search the application log for `Decryption failed` to see the raw encrypted link.

**Fix**: If the format changed, the AppX decryption logic (AES key/IV constants) needs updating.

---

### `mp4decrypt` exits with error code 1

**Cause**: The Widevine key is wrong or the media file is corrupted.

**Fix**:
1. Verify the `Key` column in your CSV is correct for that content
2. Re-run extraction to get a fresh key
3. Check the application log for `mp4decrypt` output

---

### Downloads succeed but S3 upload fails

**Cause**: AWS permission issue or network timeout.

**Diagnosis**: Search the application log for `S3UploadFailedError` or `Upload failed`.

**Fix**:
1. Verify IAM permissions (`s3:PutObject` on the target bucket)
2. Check the AWS region in `.env` matches the bucket's region
3. The item will be in the errors CSV — retry with Option 6

---

## AWS / S3 Issues

### `NoCredentialsError` or `InvalidSignatureException`

**Cause**: Malformed or expired AWS credentials.

**Fix**:
1. Verify `.env` has correct `AWS_ACCESS_KEY` and `AWS_SECRET_KEY`
2. In AWS Console: IAM → Users → your user → Security credentials → check/rotate keys
3. Ensure `AWS_REGION` matches the bucket's region

---

### Bucket creation fails: `BucketAlreadyExists`

**Cause**: Bucket names are globally unique across all AWS accounts.

**Fix**: Choose a more unique bucket name (e.g., prefix with your organization name).

---

### Option 10 (S3 URL check) shows many items as "Not Uploaded"

**Cause**:
- Upload failed silently (check the application log)
- Wrong bucket selected
- Objects uploaded to a different key path

**Fix**:
1. Run Option 6 with the original CSV (dedup will skip already-uploaded items)
2. Or use the errors CSV if one was generated during the upload run

---

## Performance Issues

### Extraction is very slow

**Cause**: Low batch size, or the platform is rate-limiting.

**Fix**:
- Increase batch size (Option 1: "Number of courses to extract together")
- Check the application log for 429 rate-limit responses — if present, reduce parallelism

---

### Upload stalling at 5+ workers

**Cause**: Platform rate-limiting or S3 throttling.

**Fix**: Reduce parallel workers to 3. For very large batches (1000+ items), 3–5 workers is the sweet spot.

---

## Build Issues

### Nuitka: `ERROR: No C compiler found`

**Fix**:
1. Install MinGW-w64 from [winlibs.com](https://winlibs.com/)
2. Choose: `GCC 13.x+ / UCRT / 64-bit / without LLVM`
3. Add the `bin/` directory to your system `PATH`
4. Verify: `gcc --version`

---

### Build completes but the executable crashes on startup

**Common causes**:
1. Missing external binary — ensure the bundled third-party binaries (ffmpeg, HLS downloader, mp4decrypt, yt-dlp) are alongside the executable
2. `.env` not present next to the executable — credentials must be in `.env` in the same directory

---

## Logs

The application writes a rotating log (5 MB × 5 backups). To investigate an issue, search the log for `ERROR` / `CRITICAL`, `Download failed`, `Decryption failed`, or `Upload failed`.

---

## Still stuck?

1. Check the full application log for the exact error
2. Search existing GitHub issues
3. Open a new issue with:
   - OS version
   - Python version (`python --version`)
   - Full error traceback (with any tokens/credentials removed)
   - Steps to reproduce
