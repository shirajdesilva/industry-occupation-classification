# WorkCover Classification Pipeline

Classifies Australian employees for workers' compensation (WorkCover) purposes. Takes free-text job titles, maps them to ANZSCO occupation codes via embedding similarity, then applies state-based routing logic to determine whether industry or occupation drives the WorkCover premium.

## What This Repo Does (Current State)

### 1. Source Data Preparation

Jupyter notebooks that convert raw reference files (Excel, DOCX) into clean Parquet/CSV:

| Notebook | Source | Output | Rows |
|---|---|---|---|
| `ANZSCO/ANZSCO.ipynb` | ABS ANZSCO 2022 Excel | `ANZSCO.parquet` (Code, Title) | 1,434 |
| `ANZSIC/ANZSIC.ipynb` | ABS ANZSIC 2006 Excel | `ANZSIC.parquet` (4-level hierarchy) | 829 |
| `OSCA/OSCA.ipynb` | OSCA Excel | `OSCA_Category_Descriptions.parquet` | 1,157 |
| `States/SA/SA SAIC.ipynb` | SA premium rates DOCX | `industry_premium_rates_2025-26.parquet` | 528 |
| `States/WA PRC/WA PRC.ipynb` | WA PRC Excel | `WA_PRC.parquet` | 517 |

### 2. Employee Classification Pipeline

`Ingest_employees_data.ipynb` runs the end-to-end classification:

1. **Ingest** — loads 500-row synthetic employee dataset + ANZSCO taxonomy
2. **ANZSCO classification** — embeds job titles and ANZSCO titles using `sentence-transformers/all-MiniLM-L6-v2` (384-dim, CPU), matches by cosine similarity
3. **State routing** — applies WorkCover rules:
   - NSW + labour hire → classification driven by worker's **occupation** (ANZSCO)
   - All other cases → classification driven by **industry** (employer or host employer ANZSIC)
4. **Export** — outputs `Output Data/workcover_classified.parquet` and `.xlsx`

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

- `Readme/classification-method-by-state.md` — how each Australian state classifies workers for WorkCover, including the critical labour hire distinction
- `Readme/workcover-classifier-project-plan.md` — full project vision and phased development plan
