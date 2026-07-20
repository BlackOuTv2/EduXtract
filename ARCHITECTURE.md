# Architecture — EduXtract

> This document describes the system design at a conceptual level. EduXtract is closed-source; component names below refer to logical responsibilities, not implementation files.

## System Overview

EduXtract is organized as a layered async CLI application. Each layer has a single responsibility; data only flows downward.

```
┌───────────────────────────────────────────────────────┐
│  Layer 1 — Presentation                               │
│  Interactive CLI: app manager + async input handling  │
│  Colorama-powered terminal UI, centered menus         │
└────────────────────────┬──────────────────────────────┘
                         │ dispatch
┌────────────────────────▼──────────────────────────────┐
│  Layer 2 — Extraction                                 │
│  ClassPlus · AppX · Graphy async API clients          │
│  Per-platform enumeration → write structured CSV      │
└────────────────────────┬──────────────────────────────┘
                         │ CSV
┌────────────────────────▼──────────────────────────────┐
│  Layer 3 — Processing                                 │
│  Reads CSV, detects platform mode, dispatches handler │
│  Manages async semaphore pool (1–20 workers)          │
└────────────────────────┬──────────────────────────────┘
                         │ per-item
┌────────────────────────▼──────────────────────────────┐
│  Layer 4 — Handlers                                   │
│  Per-platform auth resolution + download request build │
└────────────────────────┬──────────────────────────────┘
                         │ download
┌────────────────────────▼──────────────────────────────┐
│  Layer 5 — Download Engine                            │
│  yt-dlp · N_m3u8DL-RE · pytubefix · direct aiohttp    │
│  mp4decrypt for Widevine-protected content            │
└────────────────────────┬──────────────────────────────┘
                         │ local file
┌────────────────────────▼──────────────────────────────┐
│  Layer 6 — Upload                                     │
│  boto3 S3 · dedup check · bucket management           │
└───────────────────────────────────────────────────────┘
```

---

## Component Diagram

```mermaid
graph LR
    subgraph CLI ["Presentation Layer"]
        M[Interactive CLI]
        AM[App Manager]
        UI[Input Handler]
        M --> AM
        M --> UI
    end

    subgraph EXT ["Extraction Layer"]
        CP[ClassPlus Extractor]
        AX[AppX Extractor]
        GR[Graphy Client]
    end

    subgraph PROC ["Processing Layer"]
        CSV[Processing Engine]
        CM[CSV Manager]
    end

    subgraph HDL ["Handler Layer"]
        CH[ClassPlus Handler]
        AH[AppX Handler]
        GH[Graphy Handler]
        PH[PDF Handler]
        NH[Direct-URL Handler]
    end

    subgraph DL ["Download Layer"]
        DW[Download Engine]
        YT[yt-dlp]
        M3[N_m3u8DL-RE]
        MD[mp4decrypt]
        FF[ffmpeg]
    end

    subgraph UP ["Upload Layer"]
        S3[S3 Uploader]
        BT[boto3]
    end

    subgraph UTL ["Shared Utilities"]
        NET[Async HTTP Session]
        CRY[Crypto]
        TKN[Token Store]
        LOG[Logger]
        SEC[Licensing]
    end

    AM --> CP & AX & GR
    AM --> CSV
    CP & AX & GR --> CM
    CM --> CSV
    CSV --> CH & AH & GH & PH & NH
    CH & AH & GH & PH & NH --> DW
    DW --> YT & M3 & MD & FF
    DW --> S3
    S3 --> BT

    AM & CP & AX & GR & CSV -.-> LOG
    CH & AH & GH -.-> NET
    AX & GH -.-> CRY
    AM -.-> TKN
    M -.-> SEC
```

---

## Async Concurrency Model

```mermaid
graph TD
    A[Processing Engine] -->|asyncio.Semaphore N| B[Worker Pool]
    B --> W1[Worker 1]
    B --> W2[Worker 2]
    B --> W3[Worker N]

    W1 --> D1[download item]
    W2 --> D2[download item]
    W3 --> D3[download item]

    D1 --> U1[upload to S3]
    D2 --> U2[upload to S3]
    D3 --> U3[upload to S3]

    U1 & U2 & U3 --> R[results collector]
    R --> E[errors CSV for failed items]
    R --> S[success summary]
```

The worker count is user-configurable (1–20). Each worker:
1. Acquires a semaphore slot
2. Resolves the download URL (may require platform auth)
3. Delegates to the download engine (subprocess or aiohttp)
4. Calls the S3 uploader on success
5. Logs to the errors CSV on failure
6. Releases semaphore

---

## Platform Extraction Flows

### ClassPlus

```mermaid
sequenceDiagram
    participant CLI
    participant CP as ClassPlus API
    participant TM as Token Store
    participant CM as CSV Manager

    CLI->>TM: load token (per bucket)
    TM-->>CLI: access token (or none)
    CLI->>CP: validate token
    CP-->>CLI: valid / invalid

    alt token invalid
        CLI->>CP: refresh with refresh token
        CP-->>CLI: new access token
        CLI->>TM: persist new token
    end

    CLI->>CP: enumerate courses
    CP-->>CLI: [course id, title, ...]

    loop per course (batch parallel)
        CLI->>CP: fetch modules
        CP-->>CLI: module list
        loop per module
            CLI->>CP: fetch content (videos/PDFs/tests)
            CP-->>CLI: signed url, key, type
            CLI->>CM: log(source url, target url, key, type)
        end
    end
```

### AppX

```mermaid
sequenceDiagram
    participant CLI
    participant AX as AppX CMS API
    participant CRY as Crypto

    CLI->>AX: fetch courses (session cookie)
    AX-->>CLI: course list

    loop per course
        CLI->>AX: fetch course content
        AX-->>CLI: encrypted link, content type

        alt encrypted video/PDF
            CLI->>CRY: decrypt link (AES-CBC)
            CRY-->>CLI: cleartext URL
        end

        CLI->>CM: log(url, target path, type)
    end
```

---

## Token & Auth Architecture

```mermaid
graph LR
    subgraph Storage ["Local Storage (gitignored)"]
        J[Per-bucket token store]
        C[Session cookie file]
    end

    subgraph ClassPlus
        AT[access token]
        RT[refresh token]
        AT & RT --> J
    end

    subgraph AppX
        CI[session cookie]
        CI --> C
    end

    subgraph Graphy
        GM[email + password]
        GS[session — in memory only]
        GM --> GS
    end

    J --> TM[Token Store]
    C --> HLP[Session Helper]
    TM & HLP & GS --> HDL[Handlers]
```

---

## Logging Architecture

```mermaid
graph LR
    src[Any module] -->|info / debug / error| L[Central Logger]
    L --> FH[Rotating File Handler\n5MB x5 backups]
    L --> CH[Stream Handler\ncolored console]
    FH & CH --> F[Noise Filter\nsilences Windows asyncio resets]
```

Log format:
```
2026-07-20 14:23:01 | INFO     | EduXtract | Target Bucket: <bucket-name>
```

---

## Build System

```mermaid
flowchart LR
    SRC[Python source\n+ dependencies] --> NK[Nuitka compiler\nstandalone · onefile · LTO]
    BIN[External binaries\nffmpeg · yt-dlp · N_m3u8DL-RE · mp4decrypt] --> NK
    NK --> EXE[Self-contained\nnative Windows binary]
```

Nuitka compiles Python to C then to a native executable. With link-time optimization, this also provides code-level hardening that makes reverse engineering significantly harder than a bytecode-based bundle.

---

## Technology Stack

| Category | Technology | Purpose |
|---|---|---|
| Language | Python 3.10+ | Core runtime |
| Async I/O | asyncio + aiohttp | Non-blocking HTTP and file I/O |
| Cloud | boto3 + botocore | AWS S3 operations |
| DRM | pywidevine + mp4decrypt | Widevine key extraction and decryption |
| Cryptography | pycryptodome | AES-CBC decryption |
| Auth | PyJWT | JWT parsing and validation |
| Data | pandas | Structured CSV processing |
| Video | yt-dlp + N_m3u8DL-RE + ffmpeg | HLS and YouTube download |
| Logging | Python logging + colorama | Rotating file + colored terminal |
| Build | Nuitka | Native binary compilation |
| Database | aiosqlite | Local SQLite (async) |
| HTTP | requests + fake-headers | Sync fallback + user-agent rotation |
