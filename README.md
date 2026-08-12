<div align="center">

<h1>🔥 MergeForge v2</h1>

**Self-hosted LLM model merging — entirely from your browser.**

Combine open-weight language models without writing a single line of code.  
Profile your hardware, pick your models, hit merge, download the result.  
No GPU? CPU-only mode is a first-class citizen.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/)
[![React 18](https://img.shields.io/badge/react-18.3-61DAFB.svg)](https://reactjs.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688.svg)](https://fastapi.tiangolo.com/)
[![MongoDB](https://img.shields.io/badge/MongoDB-7.0-47A248.svg)](https://www.mongodb.com/)

[What is MergeForge?](#-what-is-mergeforge) · [Features](#-features) · [Installation](#-installation) · [Quick Start](#-quick-start) · [API Reference](#-api-reference) · [Roadmap](#-roadmap)

</div>

---

## What is MergeForge?

The top open-weight models on the HuggingFace leaderboards are **merges of merges of merges** — but the tooling that makes that possible has always lived at the Python power-user level. Notebooks, YAML files, command-line incantations. MergeForge tears down that wall.

It's a production-grade, self-hosted web application that wraps the [`mergekit`](https://github.com/cg123/mergekit) library and adds everything it's missing: a real UI, hardware awareness, real-time progress, automatic quality evaluation, GGUF compression, and multi-user rate limiting. You open a browser, pick models, click merge, and walk away.

```
Blend a coding model + a reasoning model → download a 3× smaller GGUF, ready for llama.cpp
```

---

## Why MergeForge?

Every other option asks you to discover problems the hard way. MergeForge tells you before you start.

| The pain | What other tools do | What MergeForge does |
|---|---|---|
| "Do I have enough RAM?" | You find out 30 min into a crash. | Hardware profiled at boot — impossible merges are hidden before you start. |
| "Will this take 5 min or 5 hours?" | Vague README guesses. | Honest ETA per tier based on your actual CPU/GPU/RAM. |
| A 7B merge silently hangs | The process freezes; you SSH in to investigate. | Stall watchdog + auto-retry + cache cleanup — fails loud with a real error, never hangs. |
| "Is the merged model any good?" | You manually prompt-test it and eyeball the results. | Automatic perplexity score (0–100) + 3 inference probes on every completed merge. |
| A 14 GB merged model to share | You upload raw safetensors and hope. | Built-in GGUF Q4_K_M export via llama.cpp — typically 3× smaller, works with llama.cpp / ollama / LM Studio. |
| One person abusing a shared server | Single-user notebook. | Tier-based daily rate limits (free / pro / enterprise) with admin override. |

---

## Features

### Merging Engine

- Wraps **mergekit** — the de-facto open-source LLM merging library
- Merge methods: `linear`, `slerp`, `ties`, `dare_ties`, `passthrough`
- Arbitrary per-source weight and density configuration
- Hard timeout per attempt (stall watchdog) + 2-hour absolute cap
- Auto-retry on transient failures (lazy_unpickle bug, OOM, partial downloads)
- HF cache hygiene — orphaned `.incomplete` files swept before and after every job

### Post-Merge Quality Evaluation

- Perplexity score computed on a built-in validation corpus
- 3 inference probes for coherence checks
- `quality_score` (0–100) with human-readable summary stored on every job
- Colour-coded in the UI: 🟢 > 80 · 🟡 > 60 · 🔴 < 60

### GGUF Compression

- Automatic **GGUF Q4_K_M** export via `llama.cpp` convert + quantize after every successful merge
- Both formats (SafeTensors + GGUF) downloadable from one page
- Failure-safe — a broken GGUF export never blocks or invalidates the merge itself

### Tier-Based Access Control

| Tier | Daily merges | Intended for |
|---|---|---|
| `free` | 3 / day | Hobbyist, evaluation |
| `pro` | 20 / day | Power user, small team |
| `enterprise` | Unlimited | Organisation-wide deployment |

- Token-based auth via a 30-word mnemonic — no passwords, no email required
- Admin endpoint to change any user's tier via `X-Admin-Secret` header
- Daily counters reset at UTC midnight

### Hardware Awareness

- Auto-detects CPU cores, RAM, swap, VRAM (via nvidia-smi), and free disk
- Maps the machine to one of four tiers (CPU-only → Ultra-scale)
- Incompatible models are hidden in the catalog with a clear explanation, not surfaced as runtime errors
- Live resource monitor on the dashboard

### Public Leaderboard

- Top 10 merges ranked by automated quality score
- Per-job `is_public` toggle (private by default)
- No auth required to read — ideal for community discovery

### Everything Else

- All subprocesses run in their own process group → cancellable from the UI
- Background async worker keeps the API responsive while a merge runs
- Restart-resilient: queued jobs re-enqueued on boot; stale "running" states automatically marked failed
- CORS, env-variable-driven config, MongoDB indexes on all hot paths

---

## Comparison

| Capability | **MergeForge** | mergekit CLI | LM Studio | Axolotl | HF AutoTrain |
|---|:---:|:---:|:---:|:---:|:---:|
| Browser UI for merging | ✅ | ❌ | ❌ | ❌ | ⚠️ paid |
| Hardware-aware model filtering | ✅ | ❌ | ⚠️ | ❌ | ❌ |
| Real-time progress + stall watchdog | ✅ | ❌ | — | ❌ | ⚠️ |
| Auto perplexity scoring | ✅ | ❌ | ❌ | ❌ | ❌ |
| Auto GGUF Q4_K_M export | ✅ | ❌ | ❌ | ❌ | ❌ |
| Multi-user with rate limits | ✅ | ❌ | ❌ | ❌ | ✅ cloud |
| Cancellable jobs from UI | ✅ | ❌ | — | ❌ | ✅ |
| Self-hosted, MIT licensed | ✅ | ✅ | ❌ | ✅ | ❌ |
| CPU-only first-class support | ✅ | ✅ | ✅ | ❌ | ❌ |

**The short version:** mergekit is the engine. MergeForge is the UI, multi-tenancy, quality scoring, compression, and deployment layer on top.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     BROWSER  (React 18)                      │
│  Landing · Auth · Dashboard · Models · Create · Jobs ·       │
│  Job Detail · Hardware · Leaderboard                         │
└─────────────────────────────┬────────────────────────────────┘
                              │  REST  (Bearer token)
┌─────────────────────────────▼────────────────────────────────┐
│                 FASTAPI BACKEND  (port 8001)                  │
│  Auth · Catalog · Validation · Job Queue · Rate Limiting ·   │
│  Worker · Quality Eval · GGUF Export · Admin · Leaderboard   │
└──────┬───────────────────────────┬──────────────────┬────────┘
       │                           │                  │
       ▼                           ▼                  ▼
  ┌─────────┐              ┌──────────────┐   ┌─────────────┐
  │ MongoDB │              │   mergekit   │   │  llama.cpp  │
  │ users,  │              │ (subprocess) │   │ convert +   │
  │  jobs   │              └──────┬───────┘   │ quantize    │
  └─────────┘                     │           └─────────────┘
                            ┌─────▼───────┐
                            │  HF Cache   │
                            │ (workspace) │
                            └─────────────┘
```

**How a merge flows through the system:**

1. `POST /api/merge/create` — validate inputs, check rate limit, enqueue job
2. Async worker picks the job and writes a mergekit YAML config
3. Each HF model is downloaded sequentially under the stall watchdog
4. `mergekit-yaml` subprocess runs; stdout is streamed in real-time to MongoDB logs
5. On success: perplexity eval → GGUF convert + quantize run as background tasks
6. HF cache is wiped, output is packaged as a tar, download links become available

---

## Installation

### Ubuntu / Debian 20.04+

```bash
# System packages
sudo apt-get update
sudo apt-get install -y python3.11 python3.11-venv python3-pip git \
                        build-essential cmake nodejs mongodb-org curl

# Yarn (frontend package manager)
sudo npm install -g yarn

# Clone
git clone https://github.com/vikrant-project/MergeForge.git
cd MergeForge

# Backend
python3.11 -m venv venv
source venv/bin/activate
pip install -r backend/requirements.txt

# Frontend
cd frontend && yarn install && yarn build && cd ..

# MongoDB
sudo systemctl enable --now mongod

# Config
cp backend/.env.example backend/.env
# Edit backend/.env — set your paths and change ADMIN_SECRET

# Start (two terminals)
cd backend && ../venv/bin/python -m uvicorn server:app --host 0.0.0.0 --port 8001
cd frontend && yarn preview --host 0.0.0.0 --port 7070
```

Open `http://localhost:7070` 🎉

---

### macOS (Ventura / Sonoma)

```bash
brew install python@3.11 node yarn mongodb-community cmake git
brew services start mongodb-community

git clone https://github.com/vikrant-project/MergeForge.git
cd MergeForge

python3.11 -m venv venv && source venv/bin/activate
pip install -r backend/requirements.txt

cd frontend && yarn install && yarn build && cd ..
cp backend/.env.example backend/.env

cd backend && ../venv/bin/python -m uvicorn server:app --host 0.0.0.0 --port 8001 &
cd frontend && yarn preview --host 0.0.0.0 --port 7070
```

> **Apple Silicon:** mergekit uses PyTorch, which has full MPS support. On M-series hardware the profiler will classify you as Tier 2 and unlock 30B merges.

---

### Kali Linux

Kali is Debian-based, so the Ubuntu steps apply. The only difference is that you need to add MongoDB's official apt repo manually first:

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

echo "deb [signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg] \
  https://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt-get update && sudo apt-get install -y mongodb-org python3.11 \
  python3.11-venv nodejs yarn build-essential cmake git

sudo systemctl enable --now mongod
# Then follow Ubuntu steps from "Clone" onward.
```

---

### Arch / Manjaro

```bash
sudo pacman -Syu --needed python python-pip nodejs yarn \
                          mongodb-bin cmake base-devel git
sudo systemctl enable --now mongodb

git clone https://github.com/vikrant-project/MergeForge.git
cd MergeForge

python -m venv venv && source venv/bin/activate
pip install -r backend/requirements.txt

cd frontend && yarn install && yarn build && cd ..
cp backend/.env.example backend/.env
# Then start backend + frontend (see Ubuntu step 8).
```

---

### Docker

```bash
git clone https://github.com/vikrant-project/MergeForge.git
cd MergeForge
docker compose up -d
```

Spins up three containers: `mongo`, `mergeforge-backend` (port 8001), and `mergeforge-frontend` (port 7070). Persistent volumes for the HF cache and merge output are configured by default.

Give the backend container as much CPU and RAM as you can. **8 GB minimum** for 1B-scale merges; **24 GB+** recommended for 7B-scale.

---

## Quick Start

1. Visit `http://localhost:7070`
2. Click **Generate signup token** → save the 30-word phrase
3. Open **Model Catalog** → pick two models marked compatible with your hardware
4. Open **New Merge** → choose a method (start with `linear`), set weights, click **Create**
5. Watch real-time logs in **Merge Jobs → [your job]**
6. On completion you'll see the quality score, the SafeTensors download, and (after ~30 s) the GGUF download
7. Toggle **Public** to enter your merge on the leaderboard at `/leaderboard`

---

## Configuration

**`backend/.env`**

```bash
MONGO_URL=mongodb://127.0.0.1:27017
DB_NAME=mergeforge
BACKEND_PORT=8001

# Point these at a disk with plenty of space — models are large
WORKSPACE_DIR=/var/lib/mergeforge/workspace
HF_CACHE_DIR=/var/lib/mergeforge/workspace/hf_cache

# Public-facing frontend URL (used for CORS and share links)
PUBLIC_BASE_URL=http://localhost:7070

# Change this to a random string before exposing the server to a network
ADMIN_SECRET=please-generate-a-random-string

# Optional: HuggingFace token for gated models
# HF_TOKEN=hf_xxx
```

**`frontend/.env`**

```bash
VITE_BACKEND_URL=http://localhost:8001
```

---

## API Reference

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/signup` | — | Create user, receive 30-word token |
| `POST` | `/api/auth/login` | — | Login with token |
| `GET` | `/api/auth/me` | Bearer | Current user including tier |
| `GET` | `/api/usage/today` | Bearer | Daily merge usage and limit |
| `GET` | `/api/models` | — | Catalog filtered by host hardware |
| `POST` | `/api/merge/validate` | Bearer | Dry-run validation + ETA |
| `POST` | `/api/merge/create` | Bearer | Enqueue a merge (rate-limited) |
| `GET` | `/api/merge/jobs` | Bearer | List your jobs |
| `GET` | `/api/merge/jobs/{id}` | Bearer | Job detail including quality score and GGUF status |
| `POST` | `/api/merge/jobs/{id}/cancel` | Bearer | Kill a running merge |
| `PATCH` | `/api/merge/jobs/{id}/visibility` | Bearer | Toggle public / private |
| `GET` | `/api/merge/jobs/{id}/download` | Token in query | Stream SafeTensors tar |
| `GET` | `/api/merge/jobs/{id}/download/gguf` | Token in query | Stream GGUF Q4_K_M |
| `GET` | `/api/leaderboard` | — | Top 10 public merges (no auth required) |
| `POST` | `/api/admin/tier` | `X-Admin-Secret` | Set a user's tier |
| `GET` | `/api/hardware/profile` | — | Static hardware tier |
| `GET` | `/api/hardware/live` | — | Live CPU / RAM / disk |
| `GET` | `/api/dashboard/stats` | Bearer | All-in-one dashboard payload |

---

## Smoke Tests

The repo ships a self-contained pipeline test that exercises the full happy path against two tiny (<200 MB) models:

```bash
cd MergeForge
venv/bin/python backend/test_pipeline.py
```

Expected output:

```
[PASS] signup creates token
[PASS] create merge accepts request
[PASS] merge job reaches terminal state in time :: final=completed
[PASS] merge job completed successfully
[PASS] output directory exists
[PASS] download endpoint returns >1MB file :: bytes=271656960
[PASS] quality_score is computed :: score=79.5 summary=Good — perplexity 33.86, 3/3 inference tests passed

=== 7 passed, 0 failed ===
```

Exit code 0 on full pass — drop it straight into CI.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError` on backend start | Venv not activated | Run `source venv/bin/activate` first |
| Frontend shows a blank page | `VITE_BACKEND_URL` wrong or CORS mismatch | Check `frontend/.env`, rebuild with `yarn build` |
| Merge stuck at 30% on 7B models | Disk pressure or OOM | Free disk space, increase swap, retry — the watchdog logs the exact cause |
| GGUF column shows "converting…" indefinitely | `llama.cpp` build failed | Install `cmake` + `build-essential`, then re-run the merge |
| Quality score never appears | Eval subprocess ran out of memory | Use a machine with 4 GB+ free RAM; `null` defaults to "Eval failed" |
| `401 Invalid token` on every request | Token expired or cleared | Re-login at `/auth` |
| Daily limit error on your first merge | UTC midnight rollover | The counter is UTC-based, not local time |

Still stuck? Check `logs/backend.err.log` — every subprocess line is mirrored there.

---

## Roadmap

- [ ] WebSocket-based live log streaming (replace polling)
- [ ] Additional merge methods: `dare_linear`, `model_stock`, `breadcrumbs`
- [ ] Direct push of merged model to HuggingFace Hub
- [ ] Multi-host distributed merging (split layers across nodes)
- [ ] In-browser chat playground to test merged models without downloading
- [ ] Stripe-backed paid tiers
- [ ] Community ratings and comments on the leaderboard

---

## Contributing

PRs welcome. Before submitting:

1. Run `backend/test_pipeline.py` — all 7 tests must pass
2. Keep components small and readable
3. Do not break the CPU-only Tier 1 path — it's the whole point

---

## License

[MIT](LICENSE) — do whatever you want with it.

---

<div align="center">

Built with 🔥 for the open-weight model community.

If MergeForge helped you ship a banger, [drop a star ⭐](https://github.com/vikrant-project/MergeForge)

</div>


<!-- Verified Co-Authored Architecture Update: 1786568146 -->
