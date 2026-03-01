# Pharma Trial Investigator · ONCO-2024-INT

**Trial Investigator · CDISC SDTM · Google ADK · Protocol 6.2**

> Synthetic data only — not for clinical use.

---

## Architecture

```
Excel Workbook (5 sheets: SDTM_RS, SDTM_AE, SDTM_LB, SDTM_EX, PROTOCOL_RULES)
        │
        ▼
┌─────────────────────────────────────────────┐
│  Data Loader  (data/data_loader.py)          │
│  Reads SDTM domains + protocol rules        │
│  into typed DataFrames (TrialData)          │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│  Rules Engine  (rules_engine/rules.py)       │
│  Deterministic protocol rules               │
│  RECIST · AE · LAB · VISIT · SAFETY         │
│  Zero AI — pure Python                      │
└────────────────┬────────────────────────────┘
                 │  fired_rules + context (structured JSON)
                 ▼
┌─────────────────────────────────────────────┐
│  ADK Agent  (agent/agent.py)                │
│  LlmAgent + 1 FunctionTool                 │
│  Model: gemini-2.5-flash (configurable)     │
│  Grounded by rules engine output            │
└────────────────┬────────────────────────────┘
                 │
                 ▼
        Output Card
   MAIN + STATUS + AI REASONING
```

**Key design principle:** The rules engine runs first and is authoritative.
The ADK agent calls `evaluate_subject()`, which invokes the rules engine, then
synthesizes a narrative. The AI cannot override or invent facts — it only
reasons on tool outputs.

---

## Project Structure

```
Pharma-trial-investigator/
├── data/
│   ├── __init__.py
│   ├── data_loader.py              # TrialData dataclass + load_trial_data()
│   └── agent_input_dataset.xlsx    # Synthetic SDTM workbook (5 sheets)
├── rules_engine/
│   ├── __init__.py
│   └── rules.py                    # RuleHit dataclass + apply_rules()
├── agent/
│   ├── __init__.py
│   └── agent.py                    # evaluate_subject tool + LlmAgent (root_agent)
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Setup

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure environment variables

Create a `.env` file in the project root:

```bash
# Path to the SDTM Excel workbook
DATASET_PATH=data/agent_input_dataset.xlsx

# Gemini model (optional — defaults to gemini-2.5-flash)
GEMINI_MODEL=gemini-2.5-flash

# Google AI Studio key (free tier)
GOOGLE_API_KEY=your-key-here
```

**Option A — Google AI Studio (free):**
Get a key at https://aistudio.google.com/app/apikey

**Option B — Vertex AI (GCP):**
```bash
export GOOGLE_GENAI_USE_VERTEXAI=true
export GOOGLE_CLOUD_PROJECT=your-project-id
export GOOGLE_CLOUD_LOCATION=us-central1
gcloud auth application-default login
```

---

## Usage

The agent is defined as `root_agent` in `agent/agent.py` and is run via the
Google ADK CLI.

### ADK Dev UI (browser)

```bash
adk web
```

Then open http://localhost:8000 and select `trial_investigator_agent`.

### ADK CLI runner

```bash
adk run agent
```

### Example prompts

```
Assess subject PT-004.
What is the RECIST classification for PT-001?
Summarise the safety signals for PT-006.
```

Subject IDs are resolved flexibly — `PT-001`, `patient 1`, and `1` all work.

---

## Subject Reference

| Subject | RECIST | Key Signals |
|---------|--------|-------------|
| PT-001 | Complete Response | SLD 62→0 mm, 3 visits, all labs normal |
| PT-002 | Partial Response (unconfirmed) | 34% reduction, 2 visits, Hgb 9.1 |
| PT-003 | Stable Disease | Small fluctuation, ongoing neuropathy, Creatinine rising |
| PT-004 | Progressive Disease | +27% from nadir + new lesion, Grade 3 SAE ×2, Hgb 7.8 |
| PT-005 | Mixed Response | 31% reduction + new lesion, ALT elevated |
| PT-006 | Partial Response (confirmed) | 40% reduction, 2 visits, no safety flags |

---

## Protocol Rules Covered

| Domain | Rules | Trigger examples |
|--------|-------|-----------------|
| RECIST | RULE-001–006 | CR/PR/SD/PD/Mixed/Unconfirmed |
| AE | RULE-007–010 | Grade ≥3 SAE, Grade 2 >28d, Ongoing |
| LAB | RULE-011–015 | Hgb <8, Creat >1.5×ULN, ALT >3×/5×ULN |
| VISIT | RULE-016–019 | Deviation >7d, Compliance <80%, Missed dose, Consecutive missed |
| SAFETY | RULE-020–022 | PD stopping, SAE notify, ≥2 reductions escalate |

---

## How `evaluate_subject` Works

1. Loads all SDTM domains via `data/data_loader.py` (`DATASET_PATH` env var required).
2. Derives context fields: `SLD_REDUCTION_PCT`, `SLD_INCREASE_FROM_NADIR`,
   `SUSTAINED_VISITS`, `CTCAE_GRADE`, `HEMOGLOBIN`, `CREATININE_ULN_RATIO`,
   `ALT_ULN_RATIO`, `DEVIATION_DAYS`, `COMPLIANCE_PCT`.
3. Calls `apply_rules()` for each domain — returns `RuleHit` objects + action strings.
4. Selects the RECIST classification (handles mixed CR/PR+PD case).
5. Returns a JSON-serializable dict — the LLM never sees raw DataFrames.

---

## Extending

**Add a new rule:** Edit `rules_engine/rules.py` — the `apply_rules()` function
reads rules from the `PROTOCOL_RULES` sheet. Add a row to the Excel sheet or
extend the rule evaluation logic in `_compare()` / `_parse_condition()`.

**Change model:** Set the `GEMINI_MODEL` environment variable, or edit
`agent/agent.py` and update the `model=` argument on `root_agent`.

**Add a new SDTM domain:** Add a sheet to the workbook, load it in
`data/data_loader.py`, derive context fields in `agent/agent.py`, and add a
`apply_rules(rules_df, domain="NEW_DOMAIN", context=context)` call.
