# State-by-State Classification Logic

This document details the regulatory logic codified into the pipeline to handle the inconsistencies in how Australian states classify workers for compensation.

## The Problem: Divergent State Rules
While all states use the ANZSIC framework as a baseline, they diverge significantly in how they apply it, particularly for **labour hire** and **staffing** scenarios. This creates a complex routing problem that manual processes often fail to navigate correctly.

## The Solution: Intelligent Regulatory Routing
The pipeline automates this decision-making process by applying the specific research-backed rules for each jurisdiction.

### 1. Standard Employers (Non-Labour Hire)
For standard employers, the solution is uniform: the **employer's predominant business activity (industry)** drives the classification, regardless of the individual worker's role. The pipeline uses the employer's ANZSIC code to map directly to state-specific WIC/SAIC/PRC codes.

### 2. Labour Hire & Staffing (The "State Split")
The pipeline applies a specialized router for labour hire employees where state rules differ:

#### **NSW: Occupation-Driven Matching**
*   **Logic:** Classification is driven by the **worker's actual occupation**.
*   **Pipeline Action:** Prioritizes the ANZSCO matching step (job title → occupation) as the primary key for WIC determination.

#### **VIC, QLD, WA, SA: Industry-Driven Matching**
*   **Logic:** Classification is driven by the **host employer's industry**.
*   **Pipeline Action:** Uses the host employer's ANZSIC code as the primary key. In WA, the pipeline specifically checks for clerical roles to apply specialized "Predominantly Clerical Staff" exceptions.

---

## Routing Summary Matrix

| State | Primary Classification Driver (Labour Hire) | Taxonomy Priority |
|---|---|---|
| **NSW** | **Worker's Occupation** | ANZSCO (Primary) |
| **VIC** | Host Employer's Industry | ANZSIC (Primary) |
| **QLD** | Host Employer's Industry | ANZSIC (Primary) |
| **WA** | Host Employer's Industry | ANZSIC (Primary) |
| **SA** | Host Employer's Industry | ANZSIC (Primary) |

---

## Technical Implementation
This logic is implemented in `Ingest_employees_data.ipynb` via a state-aware routing function that switches the classification key based on the `state` and `is_labour_hire` flags in the employee data.
