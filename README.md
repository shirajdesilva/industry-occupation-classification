# WorkCover Classification Pipeline

## The Commercial Headache: Cross-State Compliance
Classifying employees for Australian Workers' Compensation (WorkCover) is a notorious manual burden for businesses operating nationally. Mapping state-based WorkCover Industry Classification (WIC) codes to standardized national codes (like ANZSIC or ANZSCO) is messy and labor-intensive due to:

- **Fragmented Data:** Each state (VIC, NSW, QLD, WA, SA) publishes its own classification codes in inconsistent formats—ranging from 300-page PDFs and Word docs to Government Gazettes.
- **Inconsistent Regulatory Rules:** Rules vary significantly by jurisdiction. For example, NSW classifies labour hire based on the worker's *occupation*, while VIC, QLD, and WA use the *host employer's industry*.
- **Manual Mapping Risks:** Manually mapping messy job titles is slow and error-prone, leading to **premium leakage** (overpaying) or **audit non-compliance**.

## The Solution: An Automated Pipeline
This project replaces manual guesswork with a modern data workflow:

### 1. Unified Data Foundation (Source to Parquet)
We programmatically parse fragmented state source files into a clean, queryable **Parquet** data lake. Extracting unstructured data from PDFs and Word docs into a clean, columnar format provides a robust foundation for all downstream logic.

### 2. Semantic Matching (Embeddings & Vector Search)
Using **Sentence Embeddings** (`all-MiniLM-L6-v2`) to map the semantic similarity between messy job titles and standard taxonomies is a massive upgrade over brittle regex. It captures the semantic meaning of roles, even when the language varies across states.

### 3. Intelligent Regulatory Routing (LLM-Enhanced Rules)
The pipeline codifies state-specific rules directly into the flow. While embeddings handle the baseline matching, we leverage (planned) **LLMs** to interpret nuanced business rules (e.g., physical location exceptions for clerical staff) that traditional semantic search might miss.

## Commercial Value
- **Reduce Premium Leakage:** Ensure companies aren't overpaying due to incorrect or overly "safe" manual classifications.
- **Ensure Compliance:** Standardize reporting across jurisdictions to reduce audit risks and improve transparency.
- **Scale Operations:** Replace spreadsheet-heavy processes with a scalable pipeline attractive to commercial finance and HR teams.

## Technical Architecture
| Component | Technology | Why |
|---|---|---|
| **Embeddings** | `sentence-transformers` | Fast on CPU, excellent for semantic similarity. |
| **Data Processing** | `pandas` / `polars` | High-performance tabular data manipulation. |
| **Storage** | `Parquet` | Preserves schema and types; significantly faster than CSV. |
| **Logic** | Python State Router | Codifies complex regulatory routing (NSW vs. VIC/QLD). |

> **Caution:** Because these classifications impact financial premiums, accuracy is paramount. The pipeline is designed for a **human-in-the-loop** auditing mechanism to spot-check logic and catch edge cases before final output is trusted.

## Setup
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# Note: Full pipeline requires torch, transformers, pandas, and polars.
```
