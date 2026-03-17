# WorkCover Classification Pipeline

## Why This Exists

Classifying employees for workers' compensation (WorkCover) in Australia is painful for three reasons:

1. **Source files are a mess.** Each state publishes its classification codes and premium rates in different formats — Excel spreadsheets, Word documents, 300-page PDFs, Government Gazette notices. There's no single clean dataset. Just getting the data into a usable form takes significant effort.

2. **State rules are unclear and inconsistent.** Every state has its own classification system derived from ANZSIC, but with different code formats, naming conventions, and edge cases. The rules for how to classify workers — especially labour hire — vary between states and aren't always well documented. NSW classifies placed workers by occupation; VIC, QLD, and WA use the host employer's industry. Figuring this out from the source material alone is time-consuming.

3. **Manual classification is slow and expensive.** Mapping free-text job titles to the correct ANZSCO occupation code or ANZSIC industry code by hand doesn't scale. Payroll and insurance teams spend hours on this, and mistakes lead to incorrect premiums.

All three of these problems can be automated — and that's what this project does.

## What This Repo Does

### 1. Source Data Preparation → Solves the parsing mess (Problem 1)

Each state publishes classification codes and premium rates in different formats — Excel, DOCX, PDF, Government Gazettes. These notebooks wrangle each raw file into clean Parquet so the pipeline can consume them without manual data entry:

| Notebook | Source | Output | Rows |
|---|---|---|---|
| `ANZSCO/ANZSCO.ipynb` | ABS ANZSCO 2022 Excel | `ANZSCO.parquet` (Code, Title) | 1,434 |
| `ANZSIC/ANZSIC.ipynb` | ABS ANZSIC 2006 Excel | `ANZSIC.parquet` (4-level hierarchy) | 829 |
| `OSCA/OSCA.ipynb` | OSCA Excel | `OSCA_Category_Descriptions.parquet` | 1,157 |
| `States/SA/SA SAIC.ipynb` | SA premium rates DOCX | `industry_premium_rates_2025-26.parquet` | 528 |
| `States/WA PRC/WA PRC.ipynb` | WA PRC Excel | `WA_PRC.parquet` | 517 |
| `States/VIC/VIC WIC.ipynb` | VIC Government Gazette PDF | `VIC_WIC.parquet` | 519 |
| `States/NSW/NSW WIC.ipynb` | NSW icare premium rates PDF | `NSW_WIC.parquet` | 538 |
| `States/QLD/QLD WIC.ipynb` | QLD Government Gazette PDF | `QLD_WIC.parquet` | 562 |

See `Readme/source-files-guide.md` for details on where each file comes from and what format it's in.

### 2. Classification Pipeline → Solves manual classification (Problem 3)

`Ingest_employees_data.ipynb` replaces the manual process of mapping free-text job titles to ANZSCO codes. Instead of a human reading each title and looking it up, the pipeline embeds all titles and matches by cosine similarity:

1. **Ingest** — loads 500-row synthetic employee dataset + ANZSCO taxonomy
2. **ANZSCO classification** — embeds job titles and ANZSCO titles using `sentence-transformers/all-MiniLM-L6-v2` (384-dim, CPU), matches by cosine similarity
3. **State routing** — applies the codified WorkCover rules (see below)
4. **Export** — outputs `Output Data/workcover_classified.parquet` and `.xlsx`

### 3. State Routing Logic → Solves the unclear rules problem (Problem 2)

The pipeline codifies the state-by-state rules that are otherwise buried in regulatory documents:

- NSW + labour hire → classification driven by worker's **occupation** (ANZSCO)
- All other cases → classification driven by **industry** (employer or host employer ANZSIC)

See `Readme/classification-method-by-state.md` for the full research behind these rules.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

No API keys required — all embeddings are generated locally on CPU.

## Key Data Files

- `Test Data/synthetic_employees.csv` — 500-row sample with ground truth ANZSCO codes
- `Output Data/workcover_classified.parquet` — classified results with WorkCover routing

## Reference Documentation

- `Readme/source-files-guide.md` — **Problem 1 deep-dive**: where to find each state's raw classification files, what format they're in, and what needs extracting
- `Readme/classification-method-by-state.md` — **Problem 2 deep-dive**: the research behind how each state classifies workers for WorkCover, including the critical labour hire distinction
- `Readme/workcover-classifier-project-plan.md` — **Solution roadmap**: architecture, tech stack, progress, and what's left to build to automate all three problems end-to-end
