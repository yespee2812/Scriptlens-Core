# ScriptLens Core

Structural editing intelligence for screenplays: **orphan scenes**, **simulate cut**, **simulate edit**, and a **non-destructive draft workflow** — without changing the writer's original upload.

The v3 **customer product** is structure-only. Plot contradiction detection (`plot_contradiction.py`) remains in the repo for **internal CI and corpus evaluation**; it is not exposed in the web app.

**Status:** Personal project. Customer v1 features run locally (and in Docker). There are **no production users**. The core path is **deterministic NLP and graph analysis** — an LLM does not decide orphans, cuts, or edits. MiniLM is an optional bounded embedding signal for semantic edges, not a decision model. See [`docs/SCRIPTLENS_STATUS_REPORT.md`](docs/SCRIPTLENS_STATUS_REPORT.md) for gaps (cloud deploy, auth, simulate regression corpus).

**License:** [MIT](LICENSE)

---

## Quick start (Docker)

Requires [Docker](https://docs.docker.com/get-docker/). The first build downloads spaCy and MiniLM (several hundred MB).

```bash
docker build -t scriptlens .
docker run --rm -p 8000:8000 scriptlens
```

Open **http://localhost:8000**. Health check: `http://localhost:8000/api/health`.

---

## Requirements (local)

- Python **3.10+**
- Windows, macOS, or Linux

Always use the venv interpreter for commands in this repo.

---

## Setup (local)

**Windows (PowerShell):**

```powershell
python -m venv venv
venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
python -m spacy download en_core_web_sm
python scripts/precache_osd_semantic.py
```

**macOS / Linux:**

```bash
python3 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
python -m spacy download en_core_web_sm
python scripts/precache_osd_semantic.py
```

The precache step is first-run only. It stores the MiniLM model used for orphan semantic edges.

---

## Run the web app

**Windows:** `venv\Scripts\python.exe run_api.py`  
**macOS / Linux:** `venv/bin/python run_api.py`

Open **http://localhost:8000**, upload a `.fountain` or `.pdf`, then use the workspace:

- Scene list with orphan badges and summary
- **Story graph** (OSD timeline)
- **Simulate cut** / **Simulate edit** (preview only)
- **Delete scene**, **Apply edit**, **Undo draft**, **Export draft**

Sessions are in-memory and expire after **24 hours** (configurable via `SESSION_TTL_HOURS`).

---

## CLI (structure-only)

Matches the customer product scope (no contradictions):

```bash
# Full structure report
python run_scriptlens.py tests/corpus/input/drama_5scene_errors.fountain --structure-only

# Simulate removing one scene
python run_scriptlens.py tests/corpus/input/drama_5scene_errors.fountain --structure-only --simulate-cut scene_002
```

On Windows you can use the same commands with `venv\Scripts\python.exe`, or `.\run_scriptlens.ps1`.

Legacy full analysis (includes contradictions — internal use):

```bash
python run_scriptlens.py tests/corpus/input/drama_5scene_errors.fountain
```

PDF conversion (manual pipeline):

```bash
python scripts/convert_pdf_to_fountain.py path/to/script.pdf
```

---

## API (quick reference)

Base URL: `http://localhost:8000/api`

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | Liveness |
| POST | `/upload` | Upload screenplay (multipart `file`) |
| GET | `/scripts/{id}` | Script metadata |
| GET | `/scripts/{id}/orphans` | Orphan list + types |
| GET | `/scripts/{id}/orphan-graph` | Story graph data |
| GET | `/scripts/{id}/scenes/{scene_id}` | Scene body for reader |
| POST | `/scripts/{id}/simulate/cut` | `{ "scene_id": "scene_002" }` |
| POST | `/scripts/{id}/simulate/edit` | `{ "scene_id", "modified_text" }` |
| POST | `/scripts/{id}/draft/delete` | `{ "scene_id" }` |
| POST | `/scripts/{id}/draft/apply-edit` | `{ "scene_id", "modified_text" }` |
| POST | `/scripts/{id}/draft/undo` | — |
| GET | `/scripts/{id}/draft/export` | Download draft `.fountain` |

Example upload:

```bash
curl -X POST http://localhost:8000/api/upload \
  -F "file=@tests/corpus/input/drama_5scene_errors.fountain"
```

Full contracts: [`docs/ARCHITECTURE_v3_STRUCTURE.md`](docs/ARCHITECTURE_v3_STRUCTURE.md) §12.

---

## Tests and CI

```bash
# Unit + API tests (236+)
python -m pytest tests/ -q

# Orphan golden fixtures
python scripts/run_orphan_spec_eval.py

# Planted contradiction corpus (internal CI gate)
python scripts/run_corpus_batch.py --compare-ground-truth
python scripts/score_corpus_baseline.py --check --min-recall 1.0 --max-false-positives 4

# Hollywood clean benchmark (local, gitignored PDFs)
python scripts/run_clean_benchmark.py
```

GitHub Actions runs pytest, orphan spec eval, and corpus baseline on push/PR to `main`.

---

## Supported inputs (customer v1)

| Format | Web / API | Notes |
|--------|-----------|-------|
| `.fountain`, `.txt`, `.md` | Yes | Best quality |
| `.pdf` (text-based) | Yes | Auto `refined` conversion + ingest warnings |
| `.docx` | No | Converter exists; upload not wired |
| `.fdx` | No | Export to Fountain or PDF first |
| Scanned / image PDF | No | Clear error; OCR not implemented |

---

## Project layout

```text
scriptlensCore/
├── Dockerfile                 Local Docker image (web app on :8000)
├── LICENSE                    MIT
├── api/                       FastAPI routes + in-memory sessions
├── web/                       Static workspace UI
├── scriptlens_structure.py    Structure-only analysis (v3 product path)
├── orphan_scene_detector.py   Orphan scene detector (OSD)
├── osd_semantic.py            Optional semantic edges (MiniLM)
├── scene_dependency.py        Continuity graph (simulate cut/edit)
├── simulate_impact_summary.py Risk headlines for simulate UI
├── pdf_ingest.py              PDF upload metadata and errors
├── plot_contradiction.py        Internal / CI only
├── run_api.py                 Start FastAPI + web UI
├── run_scriptlens.py            CLI
├── scripts/                   Batch eval, PDF tools, benchmarks
├── tests/                     Unit and API tests
└── docs/                      Architecture, UX, status — see docs/README.md
```

---

## Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `CORS_ORIGIN` | `*` | Allowed browser origins (comma-separated in production) |
| `SESSION_TTL_HOURS` | `24` | In-memory session TTL |
| `OSD_DISABLE_SEMANTIC` | unset | Set to `1` to skip MiniLM semantic edges (tests) |

---

## Documentation

| Start here | Purpose |
|------------|---------|
| [`docs/README.md`](docs/README.md) | Full documentation index |
| [`docs/SCRIPTLENS_STATUS_REPORT.md`](docs/SCRIPTLENS_STATUS_REPORT.md) | Current status, metrics, next steps |
| [`docs/ARCHITECTURE_v3_STRUCTURE.md`](docs/ARCHITECTURE_v3_STRUCTURE.md) | v3 architecture (authoritative) |
| [`docs/UX_SPEC_v1.md`](docs/UX_SPEC_v1.md) | UX spec + shipped checklist |
| [`docs/CLIENT_PITCH_SIMULATE_FEATURES.md`](docs/CLIENT_PITCH_SIMULATE_FEATURES.md) | Demo talking points |

Corpus and benchmarks: [`tests/corpus/README.md`](tests/corpus/README.md).

---

## Not in customer v1 yet

- Production hosting (auth, persistent storage, public URL)
- User accounts, billing, persistent storage
- `.docx` upload wiring
- High-risk scene badges in web UI (computed in engine)
- Simulate regression CI scorecard (ground-truth template exists)
- Contradiction panel in web UI (by design)
- Chrome extension, `.fdx` import, scanned PDF / OCR

---

*ScriptLens — Upload your script, see loose scenes, preview what breaks if you cut or rewrite.*
