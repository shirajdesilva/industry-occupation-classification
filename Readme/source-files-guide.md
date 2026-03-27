# Unified Data Source Guide

This guide describes how the pipeline programmatically reconciles fragmented regulatory source files into a unified, queryable data foundation.

## The Problem: Data Fragmentation
Australian WorkCover data is a "parsing nightmare." Each state publishes its rates and codes in non-standardized formats:
*   **VIC & QLD:** Buried in 300+ page Government Gazette PDFs.
*   **NSW:** Split across multiple icare PDF documents (Classification vs. Rates).
*   **WA:** Provided in legacy Excel formats.
*   **SA:** Published as Word (.docx) rate schedules.
*   **TAS:** Published as a KPMG actuarial report PDF with embedded rate tables.
*   **ACT:** Published as a Finity Consulting actuarial report PDF with text-based rate listings.

## The Solution: Automated Ingestion to Parquet
We solve this by maintaining a suite of dedicated ingestion notebooks that wrangle these raw files into a clean **Parquet** lake. This ensures the main classification engine operates on a standardized schema regardless of the source format.

### Core Reference Taxonomies
These provide the semantic labels for the embedding engine:
*   **ANZSCO (ABS):** 1,400+ occupation codes and titles used for NSW labour hire and semantic job matching.
*   **ANZSIC (ABS):** 500+ industry classes used as the bridge for industry-based state classifications.

### State-Specific Ingestion Status

| Jurisdiction | Raw Source Format | Pipeline Module | Output |
|---|---|---|---|
| **VIC** | Govt Gazette (PDF) | `States/VIC/VIC WIC.ipynb` | `VIC_WIC.parquet` |
| **NSW** | icare Rates (PDF) | `States/NSW/NSW WIC.ipynb` | `NSW_WIC.parquet` |
| **QLD** | Govt Gazette (PDF) | `States/QLD/QLD WIC.ipynb` | `QLD_WIC.parquet` |
| **WA** | WorkCover (XLS) | `States/WA/WA PRC.ipynb` | `WA_PRC.parquet` |
| **SA** | RTWSA (DOCX) | `States/SA/SA SAIC.ipynb` | `industry_premium_rates.parquet` |
| **TAS** | KPMG Actuarial (PDF) | `States/TAS/TAS WIC.ipynb` | `TAS_WIC.parquet` |
| **ACT** | Finity Actuarial (PDF) | `States/ACT/ACT WIC.ipynb` | `ACT_WIC.parquet` |

## Usage for New Jurisdictions
To add a new jurisdiction (e.g., NT):
1.  Identify the raw rate schedule (usually an actuarial PDF).
2.  Create a new notebook in `States/` using the PDF-to-Pandas patterns established in the VIC or NSW modules.
3.  Export the cleaned data to `.parquet` to be consumed by the main pipeline.
