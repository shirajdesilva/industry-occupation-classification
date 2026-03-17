# WorkCover Classification Method — State-by-State Summary

## Why This Document Exists

This is the research behind **Problem 2: State rules are unclear and inconsistent.** Every state has its own classification system derived from ANZSIC, but the rules — especially for labour hire — are scattered across regulatory PDFs, Government Gazettes, and insurer documentation. I spent time digging through these sources to understand exactly how each state works so the pipeline can apply the right logic automatically. This doc captures what I found.

---

## Standard Employers (non-labour hire)

**ALL states use the same approach**: classification is based on the **employer's predominant business activity (industry)**, NOT on what individual workers do.

Example: An accountant, a receptionist, and a truck driver all working at a construction company → ALL classified under the construction industry WIC. Their individual occupations are irrelevant.

| State | Classification System | Based On | Approach |
|---|---|---|---|
| VIC | WIC (~510 codes) | ANZSIC 2006 | Employer's predominant business activity |
| NSW | WIC (538 codes) | ANZSIC (NSW variations) | Employer's predominant business activity |
| QLD | WIC (~500+ codes) | ANZSIC | Employer's predominant business activity |
| SA | SAIC (528 codes) | ANZSIC 2006 | Employer's predominant business activity |
| WA | PRC (~500+ codes) | ANZSIC 2006 (+ WA variations) | Employer's predominant business activity |
| TAS | ANZSIC class codes | ANZSIC 2006 directly | Employer's predominant business activity |
| ACT | ANZSIC class codes | ANZSIC 2006 directly | Employer's predominant business activity |
| NT | Insurer-determined | ANZSIC-based | Employer's predominant business activity |

No exceptions. No state classifies standard employers by occupation.

---

## Labour Hire Employers — HERE IS WHERE THE STATES DIFFER

This is the critical distinction. When a labour hire / staffing company places a worker with a host employer, the states diverge on HOW to classify that worker for premium purposes.

### NSW — OCCUPATION of the worker

**Source**: icare WIC System (Table B), Note 8

> "The class applicable to each category of worker hired out is the class that applies to
> the activity most closely associated with the **occupation of the worker** provided by
> the labour hire business."

**What this means**: A labour hire firm placing an accountant, a labourer, a nurse, and an electrician at the SAME construction company would classify each worker under DIFFERENT WICs based on what each worker actually does:
- Accountant → Accounting Services WIC
- Labourer → Construction WIC
- Nurse → Health Services WIC
- Electrician → Electrical Services WIC

**Exception**: Office/admin staff of the labour hire company itself (whose job is placing workers) go under WIC 786100 Employment Placement Services.

**Implication for my pipeline**: In NSW, the ANZSCO classification (job title → occupation) is the PRIMARY driver for labour hire WIC determination. I must know what the worker does.

---

### QLD — INDUSTRY of the host employer (client business)

**Source**: Queensland Government Gazette, WorkCover Queensland Labour Hire WIC Allocation Guide

> "Even though all four workers have different occupations, the wages are allocated
> according to the **industry of the client business**. Wages are not allocated according
> to the occupation of the worker provided."

**What this means**: A labour hire firm placing an accountant, a labourer, a nurse, and an electrician at the SAME construction company would classify ALL workers under the SAME construction labour hire WIC:
- Accountant → Construction labour hire WIC
- Labourer → Construction labour hire WIC
- Nurse → Construction labour hire WIC
- Electrician → Construction labour hire WIC

QLD has 20 specialised labour hire WIC codes that map to broad industry groupings of the host employer.

**Implication for my pipeline**: In QLD, the host employer's ANZSIC code is the PRIMARY driver. The worker's occupation is less relevant (though still useful context).

---

### VIC — INDUSTRY of the host employer (client's workplace)

**Source**: WorkSafe Victoria — "Labour hire: Employers and insurance premiums"

> "Your imputed workplaces are the on-hire workplaces where you send your workers to
> perform work for your labour hire client. These workplaces are classified to the **same
> industry classification as your labour hire client's workplace**."

**What this means**: Same as QLD — all workers placed at a construction site get the construction WIC, regardless of their individual occupation.

VIC uses an "imputed workplace" system where the labour hire provider registers each client's workplace with their WorkSafe Agent, and the workplace gets the client's industry classification.

**Implication for my pipeline**: In VIC, the host employer's ANZSIC code drives the classification. Worker occupation is secondary.

---

### WA — INDUSTRY of the host employer

**Source**: WorkCover WA Industry Classification Order

> Example: "A labourer, project manager, engineer and accountant are supplied to a mineral
> exploration business. The appropriate industry classification is 10120 (Mineral
> Exploration) and all of the workers' wages would be assigned and declared under [this PRC]."

**What this means**: Same as VIC and QLD — host employer's industry drives it.

**Exception**: If workers are predominantly clerical staff, they may go under PRC 72120 "Labour Supply Services – Predominantly Clerical Staff".

**Implication for my pipeline**: Host employer's ANZSIC is primary, but worker occupation matters for identifying the clerical staff exception.

---

### SA — INDUSTRY of the host employer (likely)

**Source**: ReturnToWorkSA — uses SAIC codes aligned to ANZSIC 2006.

SA follows the industry-based model. Labour hire classification rules align with the general approach of classifying by the client's industry.

---

### TAS, ACT, NT — INDUSTRY of the host employer (likely)

These smaller jurisdictions follow the general industry-based approach. NT has no gazetted rates (insurers set independently).

---

## Summary Matrix

| State | Standard Employer | Labour Hire — What Drives Classification? | Worker Occupation Matters? |
|---|---|---|---|
| **NSW** | Employer's industry | **OCCUPATION of the worker** | **YES — Primary driver** |
| **VIC** | Employer's industry | Industry of host employer | Secondary (useful context) |
| **QLD** | Employer's industry | Industry of host employer | No (explicitly stated) |
| **WA** | Employer's industry | Industry of host employer | Only for clerical exception |
| **SA** | Employer's industry | Industry of host employer (likely) | Secondary |
| **TAS** | Employer's industry | Industry of host employer (likely) | Secondary |
| **ACT** | Employer's industry | Industry of host employer (likely) | Secondary |
| **NT** | Employer's industry | Insurer-determined | Varies |

---

## How This Feeds Into the Pipeline

The research above is what drives the state routing logic in `Ingest_employees_data.ipynb`. In short: both the ANZSCO step (job title → occupation) and the ANZSIC step (employer/host industry) are essential — which one is the *primary driver* depends on the state and whether the worker is labour hire. See the main `README.md` for the pipeline overview, and `Readme/workcover-classifier-project-plan.md` for the full architecture and roadmap.

