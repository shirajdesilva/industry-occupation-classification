# Classification Pipeline Roadmap

This document outlines the architecture and phased execution for the automated WorkCover classification engine.

## Problem Context
Manual classification of employees across multiple state jurisdictions is slow, expensive, and high-risk. This roadmap transitions that process from manual spreadsheet mapping to an automated, embedding-first pipeline.

## Technical Architecture

### 1. Data Foundation (The Ingestion Layer)
*   **Goal:** Convert messy PDF/Word/Excel source files into structured Parquet.
*   **Status:** Complete for VIC, NSW, QLD, WA, and SA.

### 2. Semantic Engine (The Embedding Layer)
*   **Goal:** Use `all-MiniLM-L6-v2` to map free-text job titles to official ANZSCO occupation codes.
*   **Strategy:** Pre-compute embeddings for all 1,400+ ANZSCO titles to allow for sub-second vector search.

### 3. Regulatory Router (The Logic Layer)
*   **Goal:** Apply state-specific rules (e.g., NSW occupation-based routing) to determine the final WorkCover code and premium rate.
*   **Strategy:** A Python-based routing engine that switches its target taxonomy based on employee state and labour hire status.

### 4. LLM Enhancement (The Interpretation Layer)
*   **Goal:** Resolve complex "gray area" regulatory definitions that embeddings alone might miss.
*   **Strategy:** Use local LLMs (Llama 3 / Mistral) to interpret specific regulatory exceptions and provide high-confidence reasoning for edge-case classifications.

## Execution Roadmap

### Phase 1: Semantic Baseline (Current)
- [x] Parse all state WIC/PRC tables into Parquet.
- [x] Implement ANZSCO embedding-based matching.
- [x] Codify basic state routing logic.
- [x] Benchmarking against synthetic 500-row dataset.

### Phase 2: Accuracy & Interpretation (Next)
- [ ] **Text Normalization:** Build a pre-processor for abbreviations (e.g., "Snr" -> "Senior") and seniority prefixes.
- [ ] **Local LLM Integration:** Implement Llama-cpp-python for edge-case reasoning.
- [ ] **ABN Lookup:** Integrate Australian Business Register API for automated industry (ANZSIC) discovery.

### Phase 3: Commercial Delivery
- [ ] **Human-in-the-Loop UI:** Build a Streamlit dashboard for batch uploads and manual audit flags.
- [ ] **Confidence Scoring:** Implement a threshold-based review system (e.g., <0.8 similarity flags for human review).
