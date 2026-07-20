# Security Policy — EduXtract

## Supported Versions

| Version | Supported |
|---|---|
| Latest (main branch) | ✅ Active support |
| Older builds | ❌ No longer supported |

---

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Email: **[contact via GitHub profile]**

Include:
- Description of the vulnerability
- Steps to reproduce
- Potential impact assessment
- (Optional) Suggested fix

You will receive an acknowledgement within **48 hours** and a full response within **7 days**.

---

## Security Design

### Credential Management

- AWS credentials are loaded **exclusively from environment variables** (via `.env` + `python-dotenv`)
- No credentials are stored in source code
- Auth tokens per platform are stored in local files (excluded from version control via `.gitignore`)
- Session cookies are stored locally and excluded from version control

### Token Handling

| Platform | Token Storage | Validation |
|---|---|---|
| ClassPlus | Per-bucket local token store (gitignored) | JWT expiry checked on each run; auto-refresh via refresh token |
| AppX | Local session-cookie file (gitignored) | Session cookie — operator must refresh from browser |
| Graphy | In-memory only | Never persisted to disk |

### Time-Based Licensing

The tool includes a network-verified expiry check:
1. Queries `worldtimeapi.org` for network time
2. Falls back to `google.com` response headers
3. Falls back to system time if both fail
4. If past expiry date: displays a contextually plausible error and exits

### DRM & Legal Scope

This tool implements Widevine DRM key extraction (`pywidevine`) and AES-CBC decryption for operational content migration. Its use is restricted to:

- **Authorized platform operators** migrating their own or their clients' content
- Scenarios where the operator has **contractual rights** to the content being processed
- Jurisdictions where such migration is legally permitted

Use of this tool to access, decrypt, or redistribute content you do not own or have authorization for is:
- A violation of this project's license terms
- Potentially illegal under the DMCA (17 U.S.C. § 1201), the Computer Fraud and Abuse Act, platform Terms of Service, and equivalent laws in other jurisdictions

---

## Known Security Considerations

| Area | Risk | Mitigation |
|---|---|---|
| AWS credentials | Exposure if `.env` is committed | `.env` gitignored; startup raises an error if keys are missing |
| Auth tokens | Exposure if token files are committed | Token and session files are gitignored |
| Platform AES key | Platform-specific constant in source | Not user-controlled; a platform-level value |
| `pywidevine` | DMCA-sensitive | Usage documented; restricted to authorized operators only |
| Self-destruct on expiry | Process + file deletion on expiry | Behavior is documented; operator should maintain a valid license |

---

## Dependency Security

To audit dependencies for known CVEs:

```bash
pip install pip-audit
pip-audit -r requirements.txt
```

Or with `safety`:

```bash
pip install safety
safety check -r requirements.txt
```

---

## Secure Build

The Nuitka build compiles the project to native C with LTO enabled:
- Source code is compiled out; not recoverable from the binary as Python bytecode
- All Python modules are statically linked into the executable
- External binaries (`ffmpeg`, `yt-dlp`, `mp4decrypt`) are copied alongside but remain unmodified
