# WorkCover Classification Engine — Project Plan

## Why This Document Exists

This is the roadmap for solving all three problems:
1. **Source files are a mess** — I'm automating the parsing of each state's raw classification data into clean Parquet (see `Readme/source-files-guide.md` for where each file comes from)
2. **State rules are unclear** — I've codified the state-by-state routing logic into the pipeline (see `Readme/classification-method-by-state.md` for the research)
3. **Manual classification is slow** — the architecture below replaces manual job title → code mapping with embedding-based similarity matching

This doc covers the architecture, tech stack, what's done, and what's left to build.

---

## Project Overview

I'm building an intelligent classification system that takes messy, free-text employee job titles and automatically maps them to:
1. **ANZSCO codes** (Australian and New Zealand Standard Classification of Occupations)
2. **Workers' compensation industry/occupation codes** based on the employee's state jurisdiction

This solves a real pain point for employers, staffing agencies, insurers, and payroll providers who manually classify thousands of employees for WorkCover purposes.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Streamlit Frontend                       │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │ Single Input  │  │ CSV Batch     │  │ Results &       │  │
│  │ (job title +  │  │ Upload        │  │ Dashboard       │  │
│  │  state +      │  │               │  │                 │  │
│  │  employer     │  │               │  │                 │  │
│  │  industry +   │  │               │  │                 │  │
│  │  labour hire?)│  │               │  │                 │  │
│  └──────┬────────┘  └──────┬────────┘  └─────────────────┘  │
└─────────┼──────────────────┼────────────────────────────────┘
          │                  │
          ▼                  ▼
┌──────────────────────────────────────────┐
│          Classification Pipeline          │
│                                          │
│  1. Text Preprocessing                   │
│     (normalise, clean, expand abbrevs)   │
│                                          │
│  2. Embedding Generation                 │
│     (sentence-transformers,              │
│      all-MiniLM-L6-v2, CPU)             │
│                                          │
│  3. ANZSCO Matching                      │
│     (cosine similarity against           │
│      pre-embedded ANZSCO titles)         │
│                                          │
│  4. Pathway Router                       │
│     ┌────────────────┬─────────────────┐ │
│     │ Standard       │ Labour Hire     │ │
│     │ Employer       │ / Staffing      │ │
│     │                │                 │ │
│     │ Use employer's │ Use worker's    │ │
│     │ own ANZSIC     │ ANZSCO + host   │ │
│     │ code directly  │ employer        │ │
│     │                │ industry to     │ │
│     │                │ determine       │ │
│     │                │ ANZSIC          │ │
│     └───────┬────────┴────────┬────────┘ │
│             │                 │           │
│             ▼                 ▼           │
│  5. State WIC/Classification Mapping     │
│     (ANZSIC → state-specific code)       │
│                                          │
│  6. Confidence Scoring                   │
│     (flag low-confidence for review)     │
└──────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────┐
│          Output                           │
│  - ANZSCO code + title (occupation)      │
│  - ANZSIC code (industry)                │
│  - Confidence score (0-1)                │
│  - State WIC/classification code         │
│  - Premium rate indicator                │
│  - Review flag (if low confidence)       │
│  - Labour hire flag (if applicable)      │
└──────────────────────────────────────────┘
```

---

## Tech Stack

| Component | Technology | Why |
|---|---|---|
| Language | Python 3.11+ | Standard for data/ML |
| Embeddings | `sentence-transformers` (all-MiniLM-L6-v2) | Fast on CPU, 384-dim, excellent for semantic similarity |
| Vector similarity | `scikit-learn` (cosine_similarity) or `FAISS` (cpu) | Efficient nearest-neighbour lookup |
| Data processing | `pandas`, `polars` | Tabular data manipulation |
| Text preprocessing | `re`, custom normaliser | Handle abbreviations, typos, variations |
| Frontend | `Streamlit` | Quick to build, looks professional, easy to deploy |
| Storage | CSV/Parquet files (or DuckDB for bonus points) | Keep it simple, no DB setup needed |
| Version control | Git + GitHub | Standard collaboration and versioning |
| Deployment | Streamlit Community Cloud (free) | Free hosting, easy to deploy |

---

## Data Strategy

### 1. ANZSCO Reference Data (Source: ABS)

The Australian Bureau of Statistics publishes the full ANZSCO classification hierarchy:
- **Major Group** (1 digit) — e.g., 2 = Professionals
- **Sub-Major Group** (2 digit) — e.g., 26 = ICT Professionals
- **Minor Group** (3 digit) — e.g., 261 = Business and Systems Analysts
- **Unit Group** (4 digit) — e.g., 2611 = ICT Business and Systems Analysts
- **Occupation** (6 digit) — e.g., 261111 = ICT Business Analyst

Download from: https://www.abs.gov.au/statistics/classifications/anzsco-australian-and-new-zealand-standard-classification-occupations/latest-release

Extract all 6-digit occupation codes with their titles and any alternative titles listed. This becomes the target label set (~1,000+ occupations).

### 2. ANZSIC Reference Data (Source: ABS)

For industry classification (some states use industry codes for premium rates):
- Download from ABS: https://www.abs.gov.au/statistics/classifications/australian-and-new-zealand-standard-industrial-classification-anzsic/latest-release

### 3. State Workers' Compensation Classification Systems

The state-by-state rules and data sources are documented separately:
- **`Readme/classification-method-by-state.md`** — how each state classifies workers (standard employers vs labour hire), the critical NSW occupation-based exception, and the summary matrix
- **`Readme/source-files-guide.md`** — where to download each state's classification codes and premium rates, file formats, and parsing priority

### 4. Synthetic Employee Sample Data

I generated ~500-1,000 synthetic employee records that simulate real-world messiness:

```
Fields:
- employee_id
- raw_job_title (the messy free-text input)
- department (optional context)
- state (VIC, NSW, QLD, SA, WA, TAS, NT, ACT)
- employer_industry (ANZSIC code or description of the employer's business)
- is_labour_hire (boolean — is this worker placed via a staffing/labour hire company?)
- host_employer_industry (if labour hire = true, the ANZSIC of where they're placed)
- expected_anzsco (ground truth for evaluation — the occupation code)
- expected_anzsic (ground truth — the industry code used for WIC determination)
```

This dual-field approach (employer_industry + host_employer_industry) is critical because:
- For standard employers: the WIC is based on employer_industry
- For labour hire: the WIC is based on host_employer_industry (where the worker is placed)

**Job title variations to include (this is where the real value is):**

| Clean Title | Messy Variations |
|---|---|
| Software Engineer | Snr Software Eng, SW Developer, Senior Dev, Software Dev III, SWE |
| Registered Nurse | RN, Reg Nurse, Registered Nurse - ICU, Nurse (Registered) |
| Electrician | Sparky, Electrical Tradesperson, Electrician A Grade, Elec Tech |
| Accounts Payable Clerk | AP Clerk, Accounts Pay., A/P Officer, Creditors Clerk |
| Forklift Driver | Forklift Operator, FLT Driver, Forky, Warehouse - Forklift |
| Data Analyst | Data Analyst II, Snr Data Analyst, Analytics Specialist, DA |
| Project Manager | PM, Proj Manager, Senior PM, Project Mgr |
| Receptionist | Front Desk, Reception, Admin/Reception, Office Receptionist |
| Chef | Head Chef, Chef de Partie, Cook, Kitchen Hand/Chef |
| Civil Engineer | Civil Eng, Structural Engineer, Civil/Structural Eng |

I included a mix of:
- White collar / blue collar / trades / healthcare / hospitality
- Different seniority levels (junior, senior, lead, head of)
- Abbreviations and slang
- Combined roles ("Admin/Reception", "Driver/Warehouse")
- Misspellings
- State distribution roughly matching population (VIC ~26%, NSW ~32%, QLD ~20%, etc.)

---

## Progress

### Completed
- [x] Set up GitHub repo with README
- [x] Download and parse ANZSCO codes from ABS into Parquet (1,434 occupation codes)
- [x] Download and parse ANZSIC codes into Parquet (829 rows, 4-level hierarchy)
- [x] Parse OSCA category descriptions into Parquet (1,157 rows)
- [x] Parse SA SAIC premium rates from DOCX into Parquet (528 rows)
- [x] Parse WA PRC codes from Excel into Parquet (517 rows)
- [x] Generate synthetic employee dataset (500 rows with ground truth labels)
- [x] Pre-compute embeddings for all ANZSCO occupation titles
- [x] Build matching pipeline (embed job title → cosine similarity → top match)
- [x] Implement state-based routing (NSW labour hire → occupation; all others → industry)
- [x] Evaluate accuracy against synthetic ground truth (Precision@1: 33.4%)
- [x] Export classified results to Parquet and Excel

---

## TODO

### Improve ANZSCO classification accuracy (currently 33.4%)
- [ ] Build text preprocessor (expand abbreviations, normalise seniority prefixes, handle slashes)
- [ ] Add OSCA alternative titles to the embedding index for broader matching
- [ ] Implement confidence thresholds (high >0.85 auto-classify, medium 0.65-0.85 review flag, low <0.65 manual review)
- [ ] Add employer industry (ANZSIC) as input context to improve accuracy
- [ ] Evaluate Top-3 and Top-5 accuracy alongside Top-1

### State-specific WIC mapping
- [ ] Add WA PRC mapping layer — match effective ANZSIC code to `States/WA PRC/WA_PRC.parquet` premium rating codes
- [ ] Add SA SAIC mapping layer — match effective ANZSIC code to `States/SA/industry_premium_rates_2025-26.parquet` premium rates
- [ ] Add VIC WorkSafe WIC mapping (source data TBD)
- [ ] Add QLD WorkCover classification mapping (source data TBD)
- [ ] Add TAS WorkCover mapping (source data TBD)
- [ ] Add ACT mapping (source data TBD)
- [ ] Return state-specific premium rate / WIC code alongside the generic `workcover_code`

### ABN → ANZSIC lookup
- [ ] Integrate ABR (Australian Business Register) API to look up ANZSIC codes by ABN
- [ ] Replace hardcoded `employer_industry_anzsic` / `host_employer_industry_anzsic` from CSV with live ABN lookups
- [ ] Add caching layer so repeated ABNs don't re-hit the API
- [ ] Handle ABN lookup failures gracefully (fall back to source data values)

### Local LLM for company name → ANZSIC classification
- [ ] Use a local LLM (e.g. Llama/Mistral via `transformers` or `llama-cpp-python`) to classify employer/host employer names to ANZSIC codes
- [ ] Use as fallback when ABN is missing or ABR lookup returns no result
- [ ] Evaluate accuracy against ground-truth ANZSIC codes in the synthetic dataset
- [ ] Consider few-shot prompting with ANZSIC taxonomy descriptions for better classification

### Streamlit frontend
- [ ] Single job title input mode (job title + state → result)
- [ ] CSV batch upload mode (upload employee file → download classified results)
- [ ] Results display with confidence score colour coding and alternative matches
- [ ] Summary dashboard for batch uploads (confidence distribution, state breakdown, review flags)
- [ ] Deploy to Streamlit Community Cloud

### Other
- [ ] Implement feedback loop (user corrects a classification → improves future matches)
- [ ] DuckDB backend for query-able results
- [ ] API endpoint (FastAPI) alongside the Streamlit UI

---

## Abbreviation Dictionary (Starter)

```python
ABBREVIATIONS = {
    "snr": "senior",
    "sr": "senior",
    "jnr": "junior",
    "jr": "junior",
    "mgr": "manager",
    "eng": "engineer",
    "engr": "engineer",
    "dev": "developer",
    "admin": "administrator",
    "asst": "assistant",
    "assoc": "associate",
    "coord": "coordinator",
    "exec": "executive",
    "dir": "director",
    "vp": "vice president",
    "svp": "senior vice president",
    "cfo": "chief financial officer",
    "cto": "chief technology officer",
    "ceo": "chief executive officer",
    "hr": "human resources",
    "it": "information technology",
    "sw": "software",
    "qa": "quality assurance",
    "pm": "project manager",
    "ba": "business analyst",
    "da": "data analyst",
    "rn": "registered nurse",
    "en": "enrolled nurse",
    "gp": "general practitioner",
    "flt": "forklift",
    "mech": "mechanic",
    "elec": "electrical",
    "tech": "technician",
    "ops": "operations",
    "acct": "accountant",
    "fin": "finance",
    "mktg": "marketing",
    "comms": "communications",
}
```

---

## Evaluation Metrics

- **Top-1 Accuracy**: % of job titles where the top match is the correct ANZSCO code
- **Top-3 Accuracy**: % where the correct code is in the top 3 matches
- **Top-5 Accuracy**: % where the correct code is in the top 5 matches
- **Mean confidence score** for correct vs incorrect classifications
- **Confusion analysis**: which occupation groups are most commonly confused
- **Processing speed**: titles classified per second on CPU

Target: Top-1 accuracy of 70-80%. Top-3 of 85-90% is achievable with embeddings.
