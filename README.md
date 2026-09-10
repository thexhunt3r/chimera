# CHIMERA — Threat Intelligence & Hunting Platform
<div align="center"><img width="607" height="411" alt="CHIMERA-CTIH" src="https://github.com/thexhunt3r/chimera/blob/main/chimera_logo.png" /> </div>

```
> v2.0 · by X-hunt3r
```

CHIMERA is an end-to-end threat intelligence and threat hunting platform that lets analysts go from a threat actor name all the way to SIEM-ready detection queries in a single workflow.

## Overview

<img width="1298" height="610" alt="Chime_Fluxo" src="https://github.com/thexhunt3r/chimera/blob/main/fluxo_chimerav1.png" />

The tool integrates four modules that can run independently or chained together:

| # | Module | What it does |
|---|---|---|
| 1 | **FindAPTGroups** | Identifies and filters threat actors from the ETDA catalogue by sector, country, tool, or name. Enriches results with IoCs and intelligence from multiple external sources. Falls back to an LLM agent when ETDA has no match. |
| 2 | **FindAPTAttck** | Correlates the threat actors found in step 1 with MITRE ATT&CK tactics and techniques (Enterprise, Mobile, or ICS domains). |
| 3 | **FindSigRules** | Matches the ATT&CK techniques from step 2 against a local Sigma rules repository and converts matching rules to the target SIEM/EDR backend format. |
| 4 | **QueryGenLLM** | Uses an LLM to synthesise detection queries for techniques that have no Sigma rule, or to retry failed conversions. Requires `LLM_QUERY_GEN_ENABLED=true`. |

All modules produce portable HTML reports saved to the `findings/` directory.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [How to Use](#how-to-use)
5. [Features](#features)
6. [Architecture and Project Structure](#architecture-and-project-structure)
7. [Examples](#examples)
8. [Troubleshooting](#troubleshooting)
9. [Contributing](#contributing)

---

## Prerequisites

### Python

- **Python 3.9 or later** (3.11+ recommended)

### Required system tools

- `git` — to clone the repository and update Sigma content
- `sigma-cli` — used internally by FindSigRules for rule conversion (managed automatically on first run when located in `sigma-cli/`)

### Required Python packages

| Package | Version | Role |
|---|---|---|
| `requests` | >=2.32.3 | HTTP client for all external API calls |
| `colorama` | >=0.4.6 | Cross-platform ANSI terminal colors |
| `python-dotenv` | >=1.0.1 | Loads `.env` into environment variables |
| `stix2` | >=3.0.1 | STIX 2.1 export (optional but recommended) |
| `pyyaml` | >=6.0.2 | Parses Sigma rule YAML files |

### API keys

All API keys are optional. Features degrade gracefully when a key is absent.

| Key | Service | What breaks without it |
|---|---|---|
| `OTX_API_KEY` | AlienVault OTX | No IoC or pulse enrichment |
| `VT_API_KEY` | VirusTotal | No malware detection enrichment |
| `MISP_URL` + `MISP_API_KEY` | MISP | No MISP event enrichment |
| `OPENCTI_URL` + `OPENCTI_API_KEY` | OpenCTI | No OpenCTI actor data |
| `NVD_API_KEY` | NVD CVE | Rate-limited CVE lookups |
| `THREATFOX_API_KEY` | Abuse.ch ThreatFox | ThreatFox works anonymously; key improves rate limits |
| `MALWAREBAZAAR_API_KEY` | Abuse.ch MalwareBazaar | MalwareBazaar works anonymously |
| `MALPEDIA_API_TOKEN` | Malpedia | Anonymous read-only access still works |
| `LLM_API_KEY` | LLM provider (OpenAI, Anthropic, etc.) | LLM features disabled |

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/x-hunt3r/chimera.git
cd chimera
```

### 2. Create a virtual environment

```bash
python -m venv env
```

Activate it:

```bash
# Linux / macOS
source env/bin/activate

# Windows (PowerShell)
.\env\Scripts\Activate.ps1

# Windows (CMD)
env\Scripts\activate.bat
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Set up the environment file

```bash
cp .env.example .env
```

Open `.env` and fill in the API keys you want to use. All keys are optional — see [Configuration](#configuration) for details.

### 5. Verify the installation

```bash
python Chimera.py --help
```

Expected output:

```text
usage: chimera [-h] [--mode {groups,attack,sigrules,full}] [--actor ACTOR]
               [--sector SECTOR] [--country COUNTRY] [--tool TOOL]
               ...
Chimera — Threat Intelligence & Hunting Platform v2.0
```

---

## Configuration

CHIMERA reads all configuration from environment variables. The recommended approach is to place them in a `.env` file in the project root (loaded automatically via `python-dotenv`).

Copy the template and edit it:

```bash
cp .env.example .env
```

### Full `.env` reference

```ini
# ── Threat intelligence enrichment APIs ──────────────────────────────────────
OTX_API_KEY=                     # AlienVault OTX API key
VT_API_KEY=                      # VirusTotal API key
MISP_URL=                        # MISP base URL, e.g. https://misp.example.com
MISP_API_KEY=                    # MISP authentication key
OPENCTI_URL=                     # OpenCTI GraphQL URL
OPENCTI_API_KEY=                 # OpenCTI Bearer token
NVD_API_KEY=                     # NVD CVE API key (improves rate limits)
THREATFOX_API_KEY=               # Abuse.ch ThreatFox (anonymous access works)
MALWAREBAZAAR_API_KEY=           # Abuse.ch MalwareBazaar (anonymous access works)

# ── Malpedia ─────────────────────────────────────────────────────────────────
MALPEDIA_API_TOKEN=              # Optional; anonymous read-only access works without it
MALPEDIA_ENRICHMENT_ENABLED=true # Set to false/0/no/off to disable Malpedia

# ── LLM provider (shared by QueryGenLLM and FindAPTGroups fallback) ──────────
LLM_PROVIDER=openai              # openai | anthropic | google | ollama | azure_openai | ibm_ica
LLM_API_KEY=                     # API key for the selected provider
LLM_MODEL=gpt-4o                 # Model name
LLM_BASE_URL=                    # Custom base URL (required for ollama, azure_openai, ibm_ica)
LLM_TIMEOUT=120                  # Request timeout in seconds
LLM_MAX_RETRIES=3                # Number of retry attempts on failure

# ── QueryGenLLM module ────────────────────────────────────────────────────────
LLM_QUERY_GEN_ENABLED=false      # Set to true to enable LLM query synthesis

# ── FindAPTGroups LLM fallback ────────────────────────────────────────────────
LLM_APT_FALLBACK_ENABLED=false   # Set to true to query LLM when ETDA has no result

# ── General ───────────────────────────────────────────────────────────────────
CACHE_TTL=86400                  # Cache time-to-live in seconds (default: 24 h)
LOG_LEVEL=INFO                   # DEBUG | INFO | WARNING | ERROR
```

### Supported LLM providers

| `LLM_PROVIDER` | Notes |
|---|---|
| `openai` | Requires `LLM_API_KEY`. Uses the OpenAI Chat Completions API. |
| `anthropic` | Requires `LLM_API_KEY`. |
| `google` | Requires `LLM_API_KEY`. |
| `ollama` | Local provider. `LLM_API_KEY` is not required. Set `LLM_BASE_URL` to your Ollama endpoint. |
| `azure_openai` | Requires `LLM_API_KEY` and `LLM_BASE_URL`. |
| `ibm_ica` | IBM ICA endpoint. Requires `LLM_API_KEY` and `LLM_BASE_URL`. |

### Supported Sigma conversion backends

`qradar`, `cortex_xdr`, `crowdstrike`, `splunk`, `elastic`, `sentinel`, `opensearch`

### Output and logging

| Variable | Default | Description |
|---|---|---|
| `LOG_LEVEL` | `INFO` | Controls verbosity. Runtime logs go to `chimera.log` and stdout. |
| `CACHE_TTL` | `86400` | ATT&CK data is cached to `chimera_cache.json` for this many seconds. |
| `--output` | `findings/` | Override the output directory using the CLI flag. |

---

## How to Use

### Interactive mode

Start CHIMERA with no arguments to enter the guided interactive menu:

```bash
python Chimera.py
```

The main menu is displayed:

```text
+======================================+
|       CHIMERA -- Main Menu           |
+======================================+
|  1 -- FindAPTGroups                  |
|  2 -- FindAPTAttck                   |
|  3 -- FindSigRules                   |
|  4 -- QueryGenLLM [OFF]              |
|  5 -- Load previous search           |
|  6 -- Export STIX 2.1                |
|  0 -- Exit                           |
+======================================+
```

Each module prompts for the necessary inputs. After each module completes you are offered the option to continue to the next module in the chain.

### Batch / CLI mode

Use `--mode` to run one or more modules non-interactively:

```bash
python Chimera.py --mode <mode> [options]
```

#### Available modes

| Mode | Modules executed |
|---|---|
| `groups` | FindAPTGroups |
| `attack` | FindAPTAttck |
| `sigrules` | FindSigRules |
| `full` | FindAPTGroups → FindAPTAttck → FindSigRules → QueryGenLLM (if enabled) |

#### Full CLI reference

```
--mode      {groups,attack,sigrules,full}   Execution mode
--actor     ACTOR                           Threat actor name
--sector    SECTOR                          Sector filter (FindAPTGroups)
--country   COUNTRY                         Target country filter (FindAPTGroups)
--tool      TOOL                            Tool/malware filter (FindAPTGroups)
--technique TECHNIQUE                       ATT&CK technique ID, e.g. T1059
--backend   BACKEND                         Sigma conversion backend (splunk, elastic, …)
--level     {informational,low,medium,high,critical}
                                            Minimum Sigma rule severity level
--domain    {e,m,i}                         ATT&CK domain: e=Enterprise, m=Mobile, i=ICS
                                            (default: e)
--operator  {AND,OR}                        Logical operator for multi-field filters
                                            (default: AND)
--output    OUTPUT                          Output directory for reports
--stix                                      Export results as STIX 2.1 bundle
--llm-backend LLM_BACKEND                  LLM backend for query generation
--llm-queries                               Run QueryGenLLM after FindAPTAttck
--test-render                               Run HTML render test and exit
--debug                                     Enable debug logging
```

### FindAPTGroups — interactive search options

When running in interactive mode, module 1 supports two search sub-modes:

- **Mode 1 — Simple search**: Select a single ETDA field (actor, observed-sectors, observed-countries, tools, …) and a value. The value accepts `*` as a wildcard. For `actor` field inputs, `APT` immediately followed by digits is automatically normalised (e.g. `APT28` → `APT 28`).
- **Mode 2 — Combined search**: Enter multiple field/value pairs combined with `AND` or `OR`.

### LLM fallback (FindAPTGroups)

When `LLM_APT_FALLBACK_ENABLED=true` and ETDA returns no match, CHIMERA automatically queries the configured LLM provider for a threat actor profile. The search term is normalised to title case before being sent to the LLM (e.g. `plump spider` → `Plump Spider`). The query uses a two-phase strategy:

1. **Phase 1 — Structured schema**: The LLM is asked to return a fixed 9-key JSON schema. If all mandatory keys are present the result is accepted immediately.
2. **Phase 2 — Legacy schema**: If Phase 1 returns an incomplete response, a fallback ETDA-compatible schema is requested.

If the LLM responds with `{"_llm_no_data": true}`, Phase 2 is not attempted.

### History

Previously executed searches are saved to `chimera_historico.json`. Select **option 5** from the main menu to review and re-run past searches.

### STIX 2.1 export

Select **option 6** from the main menu, or pass `--stix` in batch mode, to export session results as a STIX 2.1 bundle. Requires the `stix2` package.

---

## Features

### FindAPTGroups

- Downloads the **ETDA Threat Group Cards** catalogue automatically.
- Filters threat actors by: actor name, observed sectors, target countries, tools/malware used.
- Supports wildcard (`*`) matching.
- Normalises actor-name inputs for ETDA searches (`APT28` → `APT 28`) using a regex (`\bAPT(\d+)\b`).
- Enriches each matching actor with data from:
  - **AlienVault OTX** — IoC indicators and pulse count
  - **CISA KEV** — Known Exploited Vulnerabilities correlated with actor tools
  - **Abuse.ch ThreatFox** — IoC indicators
  - **Abuse.ch MalwareBazaar** — malware sample hashes
  - **Ransomware.live** — recent ransomware victims attributed to the actor
  - **MISP** — events referencing the actor
  - **OpenCTI** — actor profile from OpenCTI instance
  - **Malpedia** — actor profile and malware families
- Generates a standalone **HTML report** (`FindAPTGroups_*.html`) in `findings/`.
- The HTML report contains collapsible enrichment blocks, MITRE ATT&CK mapping tables, tool/malware tables, and reference links.
- **LLM fallback**: when enabled and ETDA has no match, queries the LLM for a structured intelligence profile, normalises the response, and includes all enrichment sections in the HTML report.

### FindAPTAttck

- Accepts MITRE ATT&CK group IDs (`G0xxx`) interactively, from a file, or via `--actor` batch flag.
- Supports three ATT&CK domains: **Enterprise** (`e`), **Mobile** (`m`), **ICS** (`i`).
- Builds an ATT&CK correlation matrix from the MITRE STIX data (cached in `chimera_cache.json`).
- Generates a **techniques file** and an **HTML report** with tactic/technique breakdowns.
- Results are passed automatically to FindSigRules when modules are chained.

### FindSigRules

- Searches the local Sigma rules repository for rules matching the techniques identified by FindAPTAttck.
- Converts matching rules to the selected SIEM/EDR backend using `sigma-cli`.
- Supports filtering by severity level (`informational`, `low`, `medium`, `high`, `critical`).
- Generates an **HTML report** with the matched rules and converted queries.
- Results are passed to QueryGenLLM when that module is enabled.

### QueryGenLLM

- **Layer A — Retry failed conversions**: For techniques that have a Sigma rule but whose `sigma-cli` conversion failed, the LLM is asked to produce a syntactically correct query.
- **Layer B — Synthesise new queries**: For techniques that have no Sigma rule at all, the LLM synthesises a detection query from the technique description and context.
- Supports multiple SIEM backends via `--llm-backend` or `--backend`.
- Generates a **JSON file**, a **plain-text file**, and an **HTML report** with all generated queries.
- Disabled by default. Enable with `LLM_QUERY_GEN_ENABLED=true`.

### Input normalisation

| Input | Search type | Transformation |
|---|---|---|
| `apt28`, `APT28`, `Apt28` | ETDA actor search | Expanded to `APT 28` (regex `\bAPT(\d+)\b`) |
| `APT 28` | ETDA actor search | Left unchanged (space already present) |
| `plump spider` | LLM fallback | Converted to `Plump Spider` (title case) |
| `DarkHydrus` | LLM fallback | Left unchanged (already contains uppercase) |

### Caching

ATT&CK STIX data is cached in `chimera_cache.json`. The default TTL is 24 hours (`CACHE_TTL=86400`). Execution history is persisted in `chimera_historico.json`.

### Reporting

- Each module generates a **self-contained HTML report** with the CHIMERA logo embedded as a Base64 data URI.
- Reports are saved to `findings/` (or the path set via `--output`).
- A **consolidated final report** is generated when the session ends (exit option or end of batch).
- STIX 2.1 bundle export is available.

---

## Architecture and Project Structure

```
chimera/
├── Chimera.py                 # Entire application — all modules, helpers, and entry point
├── requirements.txt           # Python dependencies
├── .env.example               # Environment variable template
├── .env                       # Your local secrets (not committed)
├── config.ini                 # Optional additional configuration
├── tg.json                    # Local ETDA threat-group catalogue (downloaded on first run)
├── chimera_cache.json         # ATT&CK STIX data cache
├── chimera_historico.json     # Execution history
├── chimera.log                # Runtime log
├── chimera_logo.png           # Project logo
├── findings/                  # Generated HTML and JSON reports
├── sigma/                     # Sigma rules repository content
├── sigma-cli/                 # sigma-cli project used for rule conversion
└── tests/
    ├── test_find_apt_groups_llm_fallback.py   # FindAPTGroups LLM fallback tests
    └── test_plump_spider_schema.py            # Normaliser / LLM response schema tests
```

### Key sections inside `Chimera.py`

| Section / symbol | Responsibility |
|---|---|
| Lines 1–90 | Module docstring, imports, logging setup |
| Lines 91–160 | Constants: `VERSION`, `DIR_FINDINGS`, `CACHE_TTL`, API keys, LLM config, ATT&CK maps |
| `_req_get()` | HTTP GET wrapper with rate-limit retry |
| `_carregar_cache()` / `_salvar_cache()` | JSON-file cache read/write |
| `registrar_historico()` | Appends execution record to `chimera_historico.json` |
| `_normalizar_resultado_grupo()` | Normalises a threat-group dict from any schema into the canonical internal schema; resolves ~30 key-name aliases; populates `_llm_extra` for LLM-only fields |
| `_normalizar_resposta_llm_apt()` | Parses a raw LLM string into a validated dict; strips markdown fences; unwraps single-key root wrappers; enforces ETDA field types |
| `_buscar_via_agente_llm()` | Two-phase LLM query (structured schema → legacy ETDA schema); normalises input to title case |
| `_chamar_llm_raw()` | Low-level LLM HTTP dispatch; supports all configured providers; implements exponential back-off retry |
| `_gerar_html_grupos()` | Generates the FindAPTGroups HTML report; renders enrichment blocks, MITRE tables, lifecycle, C2, mitigations, references |
| `executar_find_apt_groups()` | FindAPTGroups module entry point |
| `executar_find_apt_attck()` | FindAPTAttck module entry point |
| `executar_find_sig_rules()` | FindSigRules module entry point |
| `executar_query_gen_llm()` | QueryGenLLM module entry point |
| `exibir_menu_principal()` | Renders the interactive main menu |
| `main()` | CLI argument parsing; batch dispatch; interactive loop |
| `if __name__ == "__main__"` | Entry point |

---

## Examples

### Example 1 — Find threat actors targeting the financial sector (batch)

```bash
python Chimera.py --mode groups --sector financial --operator AND
```

CHIMERA downloads the ETDA catalogue, filters actors whose `observed-sectors` contains `financial`, enriches each match, and saves an HTML report to `findings/`.

Sample output:

```text
[*] Fetching Threat Actor catalogue (ETDA)...
  -> Last database update: 2025-01-15
  -> 3 group(s) found.
  Groups found: 3
  Top 5 groups:
    1. APT38  (North Korea)
    2. Carbanak  (Unknown)
    3. FIN7  (Unknown)
[*] Enriching via OTX AlienVault...
[*] Report generated: findings/chimera_report_APTGroups_20250115_120000.html
```

---

### Example 2 — Full chain for APT28 with Splunk output (batch)

```bash
python Chimera.py --mode full --actor "APT28" --backend splunk --output ./reports/
```

This runs the complete chain:

1. **FindAPTGroups** — locates APT28 in the ETDA catalogue and enriches the result.
2. **FindAPTAttck** — retrieves all MITRE ATT&CK Enterprise techniques attributed to APT28.
3. **FindSigRules** — finds Sigma rules for each technique and converts them to Splunk SPL.
4. Saves HTML reports and a Splunk SPL file to `./reports/`.

---

### Example 3 — LLM fallback for an unknown group (interactive)

Enable the LLM fallback in `.env`:

```ini
LLM_APT_FALLBACK_ENABLED=true
LLM_PROVIDER=openai
LLM_API_KEY=sk-...
LLM_MODEL=gpt-4o
```

Then start interactive mode and search for a group not in the ETDA catalogue:

```bash
python Chimera.py
```

```
Choose [1]: 1
Search option number: 0          ← field index for "actor"
Value for 'ACTOR' (* = wildcard): plump spider
```

Since "Plump Spider" is not in ETDA, the LLM fallback activates:

```text
No groups found matching the given criteria.
[~] Activating LLM agent fallback for: 'Plump Spider'...
  [LLM] Data retrieved from LLM agent for 'Plump Spider'.
  [!] Results sourced from LLM agent — verify before operational use.
[*] Report generated: findings/chimera_report_APTGroups_20250115_120500.html
```

The report includes the full actor profile, MITRE ATT&CK mapping, tools, attack lifecycle, C2 infrastructure, mitigations, and authoritative references — all synthesised by the LLM.

---

### Example 4 — Sigma rules for a specific technique (batch)

```bash
python Chimera.py --mode sigrules --technique T1059 --backend elastic --level high
```

FindSigRules searches the Sigma repository for rules tagged `T1059` with severity `high` or above, converts them to Elasticsearch Query DSL, and saves the output to `findings/`.

---

## Troubleshooting

### `ERROR: 'requests' not installed`

```bash
pip install requests
```

### `ERROR: 'colorama' not installed`

```bash
pip install colorama
```

### `ERROR: Could not load the group catalogue`

CHIMERA could not download or parse the ETDA JSON. Check:

- Internet connectivity.
- That `https://apt.etda.or.th/cgi-bin/getcard.cgi?g=all&o=j` is reachable.
- Proxy settings if you are behind a corporate proxy (set `HTTP_PROXY` / `HTTPS_PROXY` environment variables).

### LLM fallback returns no data

- Verify `LLM_APT_FALLBACK_ENABLED=true` is set in `.env`.
- Verify `LLM_API_KEY` is set for non-Ollama providers.
- Check `chimera.log` for error messages from `_chamar_llm_raw`.
- Increase `LLM_TIMEOUT` if the provider is slow.
- For Ollama, ensure `LLM_BASE_URL` points to your local endpoint (e.g. `http://localhost:11434/api/chat`).

### `LLM APT fallback (structured): missing keys [...]`

The LLM returned an incomplete structured response. CHIMERA automatically falls back to the legacy schema. If the legacy schema also fails, check the `chimera.log` file for the raw LLM response.

### ATT&CK data seems stale

Delete `chimera_cache.json` to force a fresh download:

```bash
rm chimera_cache.json   # Linux/macOS
del chimera_cache.json  # Windows CMD
```

### `sigma-cli` not found

CHIMERA expects `sigma-cli` to be available in the `sigma-cli/` subdirectory. Clone it:

```bash
git clone https://github.com/SigmaHQ/sigma-cli.git sigma-cli
cd sigma-cli
pip install .
cd ..
```

### HTML report not opening properly

All reports are self-contained single-file HTML. Open them directly in any modern browser. The logo and all styles are embedded inline — no internet connection is required to view the report.

### `ImportError: No module named 'stix2'`

STIX 2.1 export requires `stix2`. Install it:

```bash
pip install stix2==3.0.1
```

If you do not need STIX export, this error can be ignored — the rest of CHIMERA works without it.

---

## Contributing

### Running the test suite

```bash
pip install pytest
python -m pytest tests/ -v
```

All 119 tests must pass before submitting a pull request.

### Code conventions

- **Language**: code, comments, variable names, and string literals are written in **English**.
- **Style**: follow the PEP 8 conventions already used in `Chimera.py`. Use 4-space indentation.
- **Type hints**: all new functions must include type annotations consistent with the existing `Dict`, `List`, `Optional`, `Tuple` usage from `typing`.
- **Defensive `get()`**: all dict accesses on LLM-sourced data must use `.get()` with a safe default. Do not use bare `dict[key]` on untrusted payloads.
- **Do-not-overwrite convention**: when adding new field alias mappings in `_normalizar_resultado_grupo()`, always check `if not n.get(canonical_field)` before assigning — never overwrite a field that is already populated by a higher-priority source.
- **Sentinel filtering**: pass all user-visible string values through `_sanitize_str()` or `_sanitize_list()` to discard sentinel tokens (`N/A`, `Unknown`, `None`, etc.).
- **Tests**: every new alias mapping or normaliser change must be accompanied by at least one new test case in `tests/test_plump_spider_schema.py` or `tests/test_find_apt_groups_llm_fallback.py`.

### Adding a new enrichment source

1. Add the API key constant near the top of `Chimera.py` following the `os.getenv("KEY", "")` pattern.
2. Implement a `_buscar_<source>(actor_name)` function that returns a `Dict` or `List`.
3. Call it inside `executar_find_apt_groups()` in the enrichment loop.
4. Store results in the result dict under a private key (e.g. `_source_data`).
5. Add a renderer block in `_gerar_html_grupos()` following the `enrich-bloco` / `enrich-toggle` pattern.
6. Document the new environment variable in `.env.example`.

### Submitting changes

1. Fork the repository and create a feature branch.
2. Make your changes with clear, focused commits.
3. Run `python -m pytest tests/ -v` — all tests must pass.
4. Open a pull request describing what was changed and why.
