# Roadmap — EduXtract

This document outlines the planned evolution of the project. Priorities are subject to change based on operational needs.

---

## Current State (v1.1.0)

- ✅ Full extraction pipeline: ClassPlus, AppX, Graphy
- ✅ DRM decryption (Widevine + AES-CBC)
- ✅ Batch async download + S3 upload
- ✅ Error tracking and retry
- ✅ Token management (JWT auto-refresh)
- ✅ Standalone `.exe` build via Nuitka
- ✅ 11-option interactive CLI menu

---

## Phase 2 — Stability & Testing (Q3 2026)

**Goal: make the codebase production-safe for multi-operator deployment**

| Feature | Priority | Why |
|---|---|---|
| pytest test suite with async fixtures | 🔴 High | No tests currently — breakage is only caught in production |
| Resume interrupted downloads (checkpoint) | 🔴 High | Large batches fail partway and restart from scratch |
| Configurable retry count + exponential backoff | 🔴 High | Current retry is manual (re-run the errors CSV) |
| `.env` validation on startup with clear error messages | 🟡 Medium | Current failure mode is opaque when keys are missing |
| Platform token health-check | 🟡 Medium | Validate tokens before starting a long extraction run |
| Structured JSON logging | 🟡 Medium | Enable log parsing and alerting |

---

## Phase 3 — Platform Expansion (Q4 2026)

**Goal: cover the remaining major Indian EdTech platforms**

| Platform | Status | Notes |
|---|---|---|
| Unacademy | 🔵 Planned | Large client base; API research needed |
| PW (Physics Wallah) | 🔵 Planned | High demand from academy migration clients |
| Scaler | 🔵 Planned | Tech-focused; likely well-structured API |
| Teachable | 🔵 Planned | International platform; REST API available |
| Thinkific | 🔵 Planned | International; well-documented API |

For each new platform, the standard pattern applies: a platform extractor plus a platform handler.

---

## Phase 4 — Cross-Platform & Distribution (Q1 2027)

| Feature | Priority | Why |
|---|---|---|
| Docker image (Linux/macOS support) | 🔴 High | Currently Windows-only due to native binaries |
| CLI flag mode (non-interactive `--platform ... --bucket ...`) | 🟡 Medium | Enables scripting and automation pipelines |
| Config file support (YAML) | 🟡 Medium | Pre-configure platform credentials and bucket mappings |
| GitHub Releases with versioned executable packages | 🟡 Medium | Easier distribution to authorized operators |

---

## Phase 5 — Intelligence & Observability (Q2 2027)

| Feature | Priority | Why |
|---|---|---|
| Web dashboard (FastAPI + React) | 🟢 Low | Visual job tracking for multi-academy operators |
| Real-time progress WebSocket | 🟢 Low | Live progress during large migrations |
| Migration diff report | 🟢 Low | Show what changed between extraction runs |
| Plugin architecture for new platforms | 🟢 Low | Enable community contributors to add platforms |
| Cost estimation (S3 egress + storage) | 🟢 Low | Helps operators estimate migration cost upfront |

---

## Not Planned

These are explicitly out of scope:

- **GUI desktop app** — the CLI is sufficient for operator use; web dashboard covers the visualization need
- **Mobile app** — no use case for mobile in operator workflows
- **Autonomous migration scheduler** — operators must initiate migrations; unattended automation creates operational risk
- **Browser extension** — out of scope for this architecture

---

## How to Suggest a Feature

Open a GitHub issue with the label `enhancement`. Include:
1. The use case (what job are you trying to do?)
2. Which platform or workflow is affected
3. Whether this is blocking an actual migration project
