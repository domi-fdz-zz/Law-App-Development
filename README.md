# Data Boundary (Law App)

A **local-first** tool for **preliminary review** of U.S. privacy and data-use restrictions. Describe a data-use scenario in plain language (e.g. "we want to train a commercial model on human RNA-seq data from GEO"). The tool extracts structured facts, asks you to confirm them, then produces a verdict from **deterministic rules**, backed by verifiable primary legal sources.

> ⚠️ This tool identifies **potentially applicable** laws and obligations. It is **not legal advice** and does not determine that a proposed use is lawful. Coverage is limited to modeled U.S. privacy and data-use laws; it does not cover copyright, intellectual property, or non-U.S. laws such as GDPR or PIPL. Absence of a flag is not clearance.

Current version: `0.1.0-alpha.1`

---

## Design principles

- **Rules decide the verdict, not the model.** The LLM only assists with fact extraction, narration, source discovery, and report-grounded Q&A. Applicability and the final verdict come from deterministic logic in `privacy/gate.py`.
- **Human in the loop.** Extracted facts are shown as an editable card; the assessment runs only on facts the user has confirmed or corrected.
- **Quotes must be verifiable.** Every statutory quote the model produces is checked verbatim against an official primary source (allowlisted government / legislature domains). Quotes that cannot be verified are dropped.
- **"Unknown" is first-class.** Missing critical facts yield `insufficient`, never a default pass.
- **Keys stay local.** The API key is stored with `0600` permissions in `~/.config/data-boundary/config.json` and is never returned to the browser (only a masked hint).

## How it works

```
Plain-language scenario
   │  ① extract (LLM)     → structured facts + formal restatement
   ▼
User confirms / edits the fact card
   │  ② assess
   ├─ gate       deterministic applicability + off-ramps (de-identification, contract layer, ...)
   │             + per-purpose verdict (research / commercial)
   ├─ retrieve   for each applicable law: reasoning + obligations, with citation,
   │             verbatim quote, and official source URL
   ├─ case-scan  case-specific deep scan that can surface authorities beyond the modeled set
   ├─ verify     fetch official sources and confirm each quote verbatim
   └─ narrate    case-specific explanation (cannot change the rule-decided verdict)
   ▼
Report + report-grounded chat assistant
```

Verdict levels (most to least severe): `restricted` → `requires_approval` → `conditionally_allowed` → `insufficient` → `allowed`

## Modeled laws

HIPAA, the Common Rule, FDA human-subjects regulations, GINA, FTC Act §5, COPPA, FERPA, GLBA, VPPA, CCPA/CPRA, Virginia VCDPA, Illinois BIPA, and the Washington My Health My Data Act. See [`privacy/source_registry.json`](privacy/source_registry.json) for the source registry.

## Project structure

```
.
├── __init__.py          package entry, version, local port (7788)
├── cli.py               CLI: serve / app / info
├── main.py              FastAPI app and all API endpoints
├── config.py            LLM backend settings (DeepSeek / OpenAI / Claude / Qwen / custom)
├── desktop.py           native desktop window via pywebview
├── desktop_entry.py     PyInstaller entry point
├── privacy/             current product: privacy / data-use review
│   ├── schema.py        seven-dimension assessment input (object/actor/use_case/basis/...)
│   ├── gate.py          deterministic applicability gate and verdict composer
│   ├── pipeline.py      two-phase run: extract → assess (retrieve, scan, verify, narrate)
│   ├── source_discovery.py  discovery/validation prompts, official-domain allowlist
│   └── source_registry.json legal source registry
├── core/                legacy clearance engine (router, rule engine, evidence, reports)
├── domain_packs/        legacy domain rule packs: biodata (active), genetic_testing,
│                        drug_procurement, agricultural_genomics, custom
├── models/schemas.py    data models for the legacy engine
├── data/                single-page UI (datause.html is the current main UI)
└── tests/               pytest suite
```

## Installation and running

Requires Python 3.11+. The code imports itself as the package `app` (e.g. `from app.main import app`), so clone into a directory named `app`:

```bash
git clone https://github.com/domi-fdz-zz/Law-App-Development.git app
python -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn pydantic httpx click python-multipart
pip install pywebview   # optional: native desktop window
pip install pytest      # optional: run tests
```

Run from the **parent directory** of `app`:

```bash
python -m app.cli serve        # start the web app and open http://localhost:7788/
python -m app.cli app          # run as a native desktop window (needs pywebview)
python -m app.cli info         # show version and LLM config (never the raw key)
```

### Configuring the LLM

Choose a provider and enter an API key under **Settings** in the UI, or use environment variables (precedence: env > config file > provider preset):

| Variable | Description |
|---|---|
| `CCA_LLM_ENDPOINT` | OpenAI-compatible `/chat/completions` endpoint |
| `CCA_LLM_MODEL` | model name |
| `CCA_LLM_API_KEY` | API key |
| `CCA_CONFIG_DIR` | custom config directory |

A key is required for fact extraction, narration, source discovery, and chat.

## Main API

| Method | Path | Description |
|---|---|---|
| GET | `/` | main UI (data-use review) |
| POST | `/api/datause/extract` | ① plain language → structured facts |
| POST | `/api/datause/assess` | ② confirmed facts → verdict report |
| POST | `/api/datause/scope` | pre-check assistant (helps frame the scenario, never gives a verdict) |
| POST | `/api/datause/chat` | report-grounded assistant (explains the verdict, never overrides it) |
| GET/POST | `/api/settings` | read / save LLM settings |
| POST | `/api/test_connection` | test the LLM connection |
| GET | `/health` | health check |

Legacy engine endpoints: `/compliance`, `/api/domain-packs`, `/api/assessments/normalize`, `/api/assessments`. Interactive API docs are available at `/docs` once the server is running.

## Tests

From the parent directory of `app`:

```bash
python -m pytest app/tests
```

Tests use FastAPI's `TestClient` and need no network access or real API key.
