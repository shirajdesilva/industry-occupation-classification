# WorkCover Classification Engine — Portfolio Project Plan

## Project Overview

An intelligent classification system that takes messy, free-text employee job titles and automatically maps them to:
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
| Version control | Git + GitHub | Portfolio visibility |
| Deployment | Streamlit Community Cloud (free) | Live demo link for resume/LinkedIn |

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

You'll want to extract all 6-digit occupation codes with their titles and any alternative titles listed. This becomes your "target label set" (~1,000+ occupations).

### 2. ANZSIC Reference Data (Source: ABS)

For industry classification (some states use industry codes for premium rates):
- Download from ABS: https://www.abs.gov.au/statistics/classifications/australian-and-new-zealand-standard-industrial-classification-anzsic/latest-release

### 3. State Workers' Compensation Classification Systems

#### Critical Insight: ALL States Use INDUSTRY-Based Classification

A common misconception is that some states classify by occupation and others by industry.
In fact, **every Australian state and territory classifies employers by their predominant
business activity (industry)**, not by individual worker occupations. They all derive their
systems from ANZSIC (Australian and New Zealand Standard Industrial Classification) 2006,
but each state has its own naming convention, number of codes, and premium rates.

The classification is applied to the **employer's business**, not to individual workers.
For example, a receptionist working at a construction company is classified under the
construction industry WIC — not under an "administration" code.

#### The Labour Hire Exception (Critical for This Project!)

The one major exception — and this is directly relevant to your ManpowerGroup experience —
is **labour hire / staffing companies**. In most states, labour hire firms must classify
each placed worker based on the **activity/industry of the host employer** (i.e., what the
worker actually does or where they're placed), NOT under a single "labour hire" industry code.

For example, in NSW: "The class applicable to each category of worker hired out is the class
that applies to the activity most closely associated with the occupation of the worker
provided by the labour hire business." (icare WIC System documentation)

Similarly in WA: A labourer, project manager, engineer and accountant supplied to a mineral
exploration business would all be classified under PRC 10120 (Mineral Exploration), not under
a generic labour hire code.

This means **staffing companies like ManpowerGroup must effectively do BOTH**:
1. Map the worker's occupation/job title to understand what they do
2. Determine the host employer's industry classification
3. Apply the correct state-specific WIC/classification code

This is exactly the problem your tool solves!

#### State-by-State Classification Details

| State | Regulator | Classification System | Code Name | # of Codes | Based On | Scheme Type | Premium Rates Set By |
|---|---|---|---|---|---|---|---|
| **VIC** | WorkSafe Victoria | WorkCover Industry Classification | WIC | ~510 | ANZSIC 2006 | Government-run (public underwriting) | WorkSafe, gazetted annually |
| **NSW** | icare / SIRA | Workers Compensation Industry Classification | WIC | 538 | ANZSIC (with NSW-specific variations) | Government-run (public underwriting) | icare, filed with SIRA annually |
| **QLD** | WorkCover Queensland | WorkCover Industry Classification | WIC | ~500+ | ANZSIC | Government-run (public underwriting) | Published in QLD Government Gazette |
| **SA** | ReturnToWorkSA | South Australian Industry Classification | SAIC | 528 | ANZSIC 2006 | Government-run (public underwriting) | ReturnToWorkSA, published annually |
| **WA** | WorkCover WA | Premium Rating Classification | PRC | ~500+ | ANZSIC 2006 (with WA-specific variations, adds "0" to 4-digit ANZSIC) | Privately underwritten | WorkCover WA recommends rates; private insurers set actual premiums |
| **TAS** | WorkSafe Tasmania (WorkCover Tasmania Board) | ANZSIC Class (direct) | ANZSIC | ~500+ | ANZSIC 2006 directly | Privately underwritten | WorkCover Tasmania Board publishes suggested rates; private insurers set actual premiums |
| **NT** | NT WorkSafe | Industry category (insurer-determined) | ANZSIC-based | Varies | ANZSIC (1993 still referenced in some stats) | Privately underwritten | **No gazetted rates** — insurers have full commercial independence to set rates |
| **ACT** | WorkSafe ACT | ANZSIC Class | ANZSIC | ~500+ | ANZSIC 2006 | Privately underwritten | Suggested reasonable rates published annually by independent actuary |

#### Key Differences Between States

**Government-run vs Private schemes:**
- **Government-run** (VIC, NSW, QLD, SA): Single insurer (the government authority). Employers must insure with the state scheme. Rates are gazetted/regulated.
- **Privately underwritten** (WA, TAS, NT, ACT): Multiple private insurers compete for business. Employers can shop around. Regulators publish recommended/suggested rates but insurers can vary.

**Classification methodology:**
- All states classify based on **predominant business activity** (what the employer mainly does)
- If an employer has multiple distinct business activities, some states allow split classifications (different WICs for different parts of the business with wages allocated accordingly)
- **Labour hire is the exception**: workers are classified by the host employer's activity / worker's occupation

**WIC code structure examples:**
- VIC WIC: 6-digit numeric codes (e.g., 692100 = Computer Consultancy Services)
- NSW WIC: 6-digit numeric codes (e.g., 786100 = Employment Placement Services)
- WA PRC: 5-digit numeric codes (ANZSIC 4-digit + "0", e.g., 69210 = Management Advice and Related Consulting Services)
- SA SAIC: Aligned to ANZSIC 2006 codes
- TAS/ACT: Use ANZSIC 2006 class codes directly (4-digit)

#### Data Sources for Each State

| State | Where to Find Classification Codes & Rates |
|---|---|
| VIC | WorkSafe Victoria — "Industry rates and key dates" page; Victorian Government Gazette (Special Gazette published annually ~May/June) |
| NSW | icare — "WIC System and Premium Rates" page; downloadable PDF of full WIC system + rates for each year (2016-17 through 2025-26 available) |
| QLD | WorkSafe QLD — "WorkCover Industry Classifications (WICs)" page; QLD Government Gazette |
| SA | ReturnToWorkSA — "Industry classifications and rates" page; SAIC premium rate schedule (DOCX download available) |
| WA | WorkCover WA — "Premium Rating Classification" page; Industry Classification Order PDF + Excel list of PRC codes with recommended rates |
| TAS | WorkSafe Tasmania — Suggested Premium Rates report (annual, KPMG-prepared actuarial report with ANZSIC-level rates) |
| NT | NT WorkSafe — No published industry rates (insurers set independently). ANZSIC used as reference but not formally gazetted. |
| ACT | WorkSafe ACT — Suggested reasonable rates by ANZSIC class published annually |

#### Implications for the Classification Tool

Given that ALL states use industry-based classification, the tool's pipeline should be:

**For standard employers:**
```
Input: Job title + State + Employer industry (ANZSIC or description)
                                    │
                                    ▼
                    ┌──────────────────────────┐
                    │ 1. Map job title → ANZSCO │  (occupation classification)
                    │    (for reporting/context)│
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ 2. Use employer's ANZSIC  │  (industry classification)
                    │    to determine state WIC │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ 3. Look up state-specific │
                    │    WIC code + premium rate│
                    └──────────────────────────┘
```

**For labour hire / staffing companies (the high-value use case):**
```
Input: Job title + State + Host employer industry
                    │
                    ▼
        ┌──────────────────────────────┐
        │ 1. Map job title → ANZSCO    │  (what the worker actually does)
        │    This MATTERS here because │
        │    it determines the WIC     │
        └──────────┬───────────────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │ 2. Determine host employer's │  (where the worker is placed)
        │    ANZSIC from occupation +  │
        │    host industry context     │
        └──────────┬───────────────────┘
                   │
                   ▼
        ┌──────────────────────────────┐
        │ 3. Map to state-specific     │
        │    WIC code + premium rate   │
        └──────────────────────────────┘
```

This dual-pathway approach is what makes the tool genuinely useful — it handles both
regular employers AND the much more complex labour hire scenario.

### 4. Synthetic Employee Sample Data

Generate ~500-1,000 synthetic employee records that simulate real-world messiness:

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

Include a mix of:
- White collar / blue collar / trades / healthcare / hospitality
- Different seniority levels (junior, senior, lead, head of)
- Abbreviations and slang
- Combined roles ("Admin/Reception", "Driver/Warehouse")
- Misspellings
- State distribution roughly matching population (VIC ~26%, NSW ~32%, QLD ~20%, etc.)

---

## Phased Development Plan

### Phase 1: Data Foundation (Days 1-3)
- [ ] Set up GitHub repo with clean README
- [ ] Download and parse ANZSCO codes from ABS into structured CSV/Parquet
- [ ] Download ANZSIC codes
- [ ] Research and compile state WorkCover classification tables (start with VIC and NSW)
- [ ] Build the ANZSCO → state WorkCover crosswalk for at least 3 states
- [ ] Generate synthetic employee dataset (500+ records with ground truth labels)

### Phase 2: Core Classification Engine (Days 4-7)
- [ ] Build text preprocessor:
  - Lowercase, strip whitespace
  - Expand common abbreviations (Snr → Senior, Eng → Engineer, Mgr → Manager, etc.)
  - Remove seniority prefixes for matching (but keep for context)
  - Handle special characters and slashes
- [ ] Pre-compute embeddings for all ANZSCO occupation titles + alternative titles
- [ ] Build the matching pipeline:
  - Embed input job title
  - Cosine similarity against ANZSCO embeddings
  - Return top-N matches with confidence scores
- [ ] Implement confidence thresholds:
  - High (>0.85): auto-classify
  - Medium (0.65-0.85): classify with review flag
  - Low (<0.65): flag for manual review
- [ ] Add state WorkCover mapping layer
- [ ] Evaluate accuracy against synthetic ground truth

### Phase 3: Streamlit Frontend (Days 8-10)
- [ ] Single job title input mode (job title + state → result)
- [ ] CSV batch upload mode (upload employee file → download classified results)
- [ ] Results display:
  - Matched ANZSCO code and title
  - Confidence score with colour coding (green/amber/red)
  - State WorkCover classification
  - Alternative matches (top 3)
- [ ] Summary dashboard for batch uploads:
  - Distribution of confidence scores
  - Count by ANZSCO major group
  - Count flagged for review
  - State breakdown

### Phase 4: Polish & Deploy (Days 11-14)
- [ ] Write proper README with:
  - Problem statement (business context)
  - Architecture diagram
  - How to run locally
  - Sample screenshots
  - Accuracy metrics
  - Future improvements
- [ ] Add evaluation notebook showing precision/recall metrics
- [ ] Deploy to Streamlit Community Cloud
- [ ] Record a 2-minute demo video (optional but impressive)

### Phase 5: Bonus Features (if time permits)
- [ ] Add employer industry (ANZSIC) as input context to improve accuracy
- [ ] Implement feedback loop (user corrects a classification → improves future matches)
- [ ] Add premium rate estimates per state
- [ ] DuckDB backend for query-able results
- [ ] dbt models for the transformation layer (signals analytics engineering skills)
- [ ] API endpoint (FastAPI) alongside the Streamlit UI

---

## Abbreviation Dictionary (Starter)

Build this out as a key preprocessing asset:

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

Track and display these in your evaluation notebook:

- **Top-1 Accuracy**: % of job titles where the top match is the correct ANZSCO code
- **Top-3 Accuracy**: % where the correct code is in the top 3 matches
- **Top-5 Accuracy**: % where the correct code is in the top 5 matches
- **Mean confidence score** for correct vs incorrect classifications
- **Confusion analysis**: which occupation groups are most commonly confused
- **Processing speed**: titles classified per second on CPU

Target: Top-1 accuracy of 70-80% is realistic and honest. Top-3 of 85-90% is very achievable with embeddings. Be transparent about limitations — that's what impresses senior hiring managers.

---

## Key Talking Points for Interviews

When discussing this project, emphasise:

1. **Business problem first**: "Employers — especially staffing companies — waste hours manually classifying employees for WorkCover. This automates 80%+ of cases and flags the tricky ones for human review."

2. **Deep domain knowledge**: "All Australian states use industry-based classification derived from ANZSIC, but each has its own system — VIC and NSW use WIC codes (~510 and 538 respectively), SA uses SAIC, WA uses PRC codes. The tool handles all of them. The critical edge case is labour hire, where classification shifts from the employer's industry to the host employer's activity — which is the scenario I dealt with daily at ManpowerGroup."

3. **Real-world messiness**: "Job titles in the wild are messy — abbreviations, slang, combined roles. A 'Sparky' is an Electrician, 'Forky' is a Forklift Operator. The preprocessing pipeline handles all of this before the embeddings do their work."

4. **Practical ML choices**: "I used sentence-transformers on CPU rather than a large LLM because in production, you need something fast and cost-effective that can classify thousands of titles in batch. The model generates embeddings once for all ANZSCO titles, then it's just cosine similarity lookups."

5. **Your unique edge**: "I built this because I saw this exact problem at ManpowerGroup — classifying thousands of contingent workers across states for WorkCover compliance. Every misclassification means either overpaying premiums or compliance risk."

6. **Honest evaluation**: "Top-1 accuracy is X%, but with the confidence threshold system, we can auto-classify high-confidence matches and route the rest for human review — which is exactly how it would work in production. The labour hire pathway is inherently harder because you need both the occupation AND the host industry to get the right WIC."

---

## GitHub Repo Structure

```
workcover-classifier/
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   ├── raw/
│   │   ├── anzsco_codes.csv
│   │   ├── anzsic_codes.csv
│   │   └── state_workcover/
│   │       ├── vic_wic_codes.csv
│   │       ├── nsw_wic_codes.csv
│   │       └── qld_classification.csv
│   ├── processed/
│   │   ├── anzsco_with_embeddings.parquet
│   │   └── state_crosswalk.csv
│   └── sample/
│       └── synthetic_employees.csv
├── src/
│   ├── __init__.py
│   ├── preprocessor.py          # Text cleaning & abbreviation expansion
│   ├── embeddings.py            # Embedding generation & caching
│   ├── classifier.py            # Core ANZSCO matching logic
│   ├── state_mapper.py          # ANZSCO → state WorkCover mapping
│   └── pipeline.py              # End-to-end classification pipeline
├── app/
│   └── streamlit_app.py         # Streamlit frontend
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   ├── 02_model_evaluation.ipynb
│   └── 03_error_analysis.ipynb
└── tests/
    ├── test_preprocessor.py
    └── test_classifier.py
```

---

## Estimated Timeline

| Phase | Effort | Calendar Time (part-time) |
|---|---|---|
| Phase 1: Data Foundation | ~8 hours | 3-4 days |
| Phase 2: Core Engine | ~12 hours | 4-5 days |
| Phase 3: Streamlit UI | ~8 hours | 3-4 days |
| Phase 4: Polish & Deploy | ~6 hours | 2-3 days |
| **Total** | **~34 hours** | **~2 weeks** |

This is very achievable alongside job searching. The key is to get Phase 1 and 2 done solidly — that's where the substance is. The UI is just the wrapper that makes it demo-able.

---

## TODO

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
