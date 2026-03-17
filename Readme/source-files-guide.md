# WorkCover Classifier — Source Files Guide

## Why This Document Exists

This is the deep-dive on **Problem 1: Source files are a mess.** There's no single clean dataset for Australian WorkCover classification — each state publishes its codes and premium rates in a different format (Excel, DOCX, PDF, Government Gazettes), from a different regulator, with different naming conventions. This doc catalogues where to find each file, what format it's in, and what needs extracting. The source data notebooks in this repo automate the parsing so I don't have to redo this manually every year when rates update.

---

## Overview

There are **3 categories** of reference data needed for this project:

| # | Category | What It Does | Source |
|---|---|---|---|
| 1 | **ANZSCO** (occupation codes) | Maps job titles to occupation codes | ABS |
| 2 | **ANZSIC** (industry codes) | Classifies employer's business activity | ABS |
| 3 | **State WIC/classification tables** | Maps ANZSIC to state-specific code + premium rate | Each state regulator |

Plus the **synthetic employee data** (already generated).

---

## File 1: ANZSCO Occupation Codes

**What:** The full list of 6-digit occupation codes with titles and alternative titles.
This is the core of the embedding/matching engine — all these titles get pre-embedded.

**Source:** Australian Bureau of Statistics (ABS)

**Download from:**
https://www.abs.gov.au/statistics/classifications/anzsco-australian-and-new-zealand-standard-classification-occupations/2022

> Go to the **"Data downloads"** tab
> Download the spreadsheets containing all occupation levels

**Important note:** ABS released **OSCA** (Occupation Standard Classification for Australia) in 2024 as a replacement for ANZSCO. However, all state WorkCover systems still reference ANZSCO 2022, so I'm **using ANZSCO 2022 for this project**. Could mention OSCA awareness as a future enhancement.

**What to extract into a CSV:**

```
anzsco_code,title,alternative_titles,unit_group,minor_group,sub_major_group,major_group,skill_level
261312,Developer Programmer,"Programmer, Software Developer, Web Developer",2613,261,26,2,1
341111,Electrician (General),"Electrician, A Grade Electrician",3411,341,34,3,3
```

**Key fields for the classifier:**
- `anzsco_code` (6-digit) — the classification target
- `title` — primary title (embed this)
- `alternative_titles` — alternative titles listed in the ABS definitions (embedding these too should massively improve matching accuracy)
- `skill_level` (1-5) — useful context for disambiguation

**Approx size:** ~1,100 individual 6-digit occupations

---

## File 2: ANZSIC Industry Codes

**What:** The full hierarchy of industry classification codes. This is the bridge between employer activity and state WIC codes — since every state's WIC system is derived from ANZSIC.

**Source:** ABS

**Download from:**
https://www.abs.gov.au/statistics/classifications/australian-and-new-zealand-standard-industrial-classification-anzsic/2006-revision-2-0

> Go to **"Related information and downloads"**
> Download "ANZSIC 2006 - Codes and Titles" (catalogue 1292.0.55.002)

**What's needed:**

```
division_code,division_title,subdivision_code,subdivision_title,group_code,group_title,class_code,class_title
A,Agriculture Forestry and Fishing,01,Agriculture,011,Mushroom and Vegetable Growing,0111,Vegetable Growing (Under Cover)
M,Professional Scientific and Technical Services,692,Management Advice and Related Consulting Services,6921,Management Advice and Related Consulting Services nec
```

**Key fields:**
- `class_code` (4-digit) — this is what state WIC systems map FROM
- `class_title` — description of the industry
- `division_code` (letter A-S) — for grouping

**Approx size:** ~506 4-digit industry classes

---

## File 3: State WIC / Classification Tables + Premium Rates

A separate file is needed for each state. Starting with VIC and NSW (biggest markets), then adding QLD, SA, WA. TAS/ACT/NT are lower priority.

### VIC — WorkSafe Victoria

**What:** WorkCover Industry Classification (WIC) codes + industry rates
**# of codes:** ~510
**Based on:** ANZSIC 2006

**Download from:**
- Industry rates: https://www.worksafe.vic.gov.au/industry-rates-and-key-dates
- Full WIC code list: Published in the **Victoria Government Gazette** (Special Gazette, published annually ~May/June)
  - 2025-26 rates: Special Gazette No. S 275, published 3 June 2025
  - These are PDFs — they'll need parsing

**File format to create:**

```
vic_wic_code,wic_description,anzsic_class_code,industry_rate_pct,year
692100,Computer Consultancy Services,6921,0.427,2025-26
301100,House Construction,3011,4.850,2025-26
```

### NSW — icare

**What:** Workers Compensation Industry Classification (WIC) codes + rates
**# of codes:** 538
**Based on:** ANZSIC (with NSW-specific variations)

**Download from:**
https://www.icare.nsw.gov.au/employers/premiums/calculating-the-cost-of-your-premium/wics-and-premium-rates

Two PDFs to download:
1. **WIC System** (the full classification table ~300 pages):
   `Workers Compensation Industry Classification System 2025-2026` (3 MB PDF)
2. **Premium Rates** (the rate for each WIC):
   `Workers compensation premium rates 2025-2026` (0.27 MB PDF)

**NSW-specific notes:**
- NSW WIC codes are 6-digit (e.g., 692100, 786100)
- NSW has a special **Labour Hire rule**: the WIC for each placed worker is based on "the activity most closely associated with the occupation of the worker"
- NSW has 17 Industry Divisions (A through Q)

**File format to create:**

```
nsw_wic_code,wic_description,division,premium_rate_pct,dust_disease_rate,year
692100,Computer Consultancy Services,L,0.343,0.010,2025-26
786100,Employment Placement Services,N,1.280,0.010,2025-26
```

### QLD — WorkCover Queensland

**What:** WorkCover Industry Classification (WIC) codes
**Based on:** ANZSIC

**Download from:**
https://www.worksafe.qld.gov.au/claims-and-insurance/workcover-insurance/premium-calculation/wic

> WIC codes published in the **Queensland Government Gazette**

### SA — ReturnToWorkSA

**What:** South Australian Industry Classification (SAIC) codes + rates
**# of codes:** 528
**Based on:** ANZSIC 2006

**Download from:**
https://www.rtwsa.com/insurance/insurance-with-us/premium-calculations/industry-classifications-and-rates

> SAIC premium rate schedule available as DOCX download (539 KB)

### WA — WorkCover WA

**What:** Premium Rating Classification (PRC) codes + recommended rates
**Based on:** ANZSIC 2006 with WA variations (adds "0" to 4-digit ANZSIC codes)

**Download from:**
https://www.workcover.wa.gov.au/resources/rates-fees-payments/premium-rating-classification/

Two files available:
1. **Industry Classification Order** (PDF, describes the PRC system and coding rules)
2. **Industry Codes for Recommended Premium Rates** (Excel .xls — the easiest source file to work with!)

**WA-specific note:** PRC codes are 5-digit. They're formed by adding "0" to the ANZSIC 4-digit code (e.g., ANZSIC 6921 -> WA PRC 69210). Some WA-specific variations exist.

### TAS — WorkSafe Tasmania

**What:** Premium rates by ANZSIC class
**Based on:** ANZSIC 2006 directly (no separate code system)

**Download from:**
https://worksafe.tas.gov.au — look for "Suggested Premium Rates" report
> Annual actuarial report (by KPMG) published as PDF with ANZSIC-level rates

### ACT — WorkSafe ACT

**What:** Suggested reasonable rates by ANZSIC class
**Based on:** ANZSIC 2006 directly

Published annually by independent actuarial analysis.

### NT — NT WorkSafe

**What:** No published industry rates — insurers set their own
**Based on:** ANZSIC (1993 still referenced in some places)

**Note:** NT is the hardest state to include because there are no gazetted rates. The regulator explicitly states that insurers have full commercial independence to set premiums. I'll likely exclude NT from the MVP and note it as a limitation.


---

## Parsing Priority & Effort

| File | Priority | Effort | Notes |
|---|---|---|---|
| ANZSCO codes | **P1 — Critical** | Medium | ABS provides spreadsheets but alt titles need extracting from definitions |
| ANZSIC codes | **P1 — Critical** | Low | ABS provides clean spreadsheet download |
| WA PRC codes | **P1 — Easy win** | Low | Already in Excel format! Starting here for state data |
| SA SAIC rates | **P1 — Easy win** | Low | Available as DOCX download |
| NSW WIC codes + rates | **P1 — Critical** | High | 300-page PDF needs parsing (considering a PDF extraction library or doing key codes manually) |
| VIC WIC rates | **P1 — Critical** | High | Government Gazette PDF needs parsing |
| QLD WIC codes | **P2** | Medium | Government Gazette PDF |
| TAS rates | **P3** | Medium | Actuarial report PDF |
| ACT rates | **P3** | Medium | Actuarial publication |
| NT | **Skip for MVP** | N/A | No published rates |

**My approach:** Start with ANZSCO + ANZSIC from ABS (clean data), then WA (Excel file), then tackle the PDFs for NSW and VIC. Can always start with a subset of WIC codes for the bigger states rather than trying to parse all 500+.

---

## The Crosswalk File

The key file I need to build myself is the **ANZSIC -> State WIC mapping**. Since most states derive their codes from ANZSIC, this is often a direct or near-direct mapping:

```
anzsic_class_code,anzsic_title,vic_wic,nsw_wic,qld_wic,sa_saic,wa_prc,tas_code,act_code
6921,Management Advice and Related Consulting Services,692100,692100,692100,6921,69210,6921,6921
3011,Residential Building Construction,301100,301100,301100,3011,30110,3011,3011
8401,Hospitals (except Psychiatric Hospitals),840100,840100,840100,8401,84010,8401,8401
```

For most codes, it's just a formatting difference (ANZSIC 4-digit -> 6-digit WIC or 5-digit PRC). But some states have splits, merges, or unique codes that don't map 1:1 — those are the ones to document carefully.
