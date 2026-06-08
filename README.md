# Industry Intelligence Framework

<p align="center">
  <img src="https://img.shields.io/badge/Analysis%20Domains-5-blue" alt="5 Domains">
  <img src="https://img.shields.io/badge/View%20Templates-25-green" alt="25 Views">
  <img src="https://img.shields.io/badge/License-MIT-yellow" alt="MIT">
</p>

> **A systematic methodology for regional industry analysis — from raw enterprise data to government-ready reports. Battle-tested on 9,000+ enterprises across 30 industries.**

---

## Why This Matters

Government agencies and industrial parks need to understand their local economy: What clusters exist? Which industries are growing? Where should investment go? Most answers come from expensive consultants with 3-month turnarounds.

This framework automates that pipeline. **5 analytical domains, 25 pre-built database views, 1 SQL query → 1 insight.** What used to take a consulting team 3 months now takes 30 seconds.

---

## The Five-Domain Model

| Domain | What It Answers | Key Methods |
|--------|----------------|-------------|
| **A: Cluster Diagnosis** | Which industries cluster where? How healthy are they? | Location Quotient (LQ), Herfindahl Index, Cluster Health Score |
| **B: Enterprise Profiling** | Who are the key players? What's their growth trajectory? | Revenue Tiers, SRDI Matrix, Foreign Capital Analysis |
| **C: Spatial Performance** | How efficiently is land being used? Where's the congestion? | Output per Hectare, Floor Area Ratio, Metro Coverage |
| **D: Supply Chain** | What are the industry chains? Who supplies whom? | Industry Chain Reconstruction, Group Network Analysis |
| **E: Innovation Ecosystem** | Where's the R&D happening? How close is innovation to production? | R&D Efficiency, Innovation-Industry Coupling, 500m Buffer Analysis |

---

## Quick Start

```sql
-- 1. Street-level competitiveness in one query
SELECT st, competitive_score, total_output, growth_rate
FROM ontology.street_competitiveness
ORDER BY competitive_score DESC;

-- 2. Which industries cluster in which streets?
SELECT industry, street, lq_score, enterprise_count
FROM ontology.street_lq
WHERE lq_score > 1.5  -- Highly specialized
ORDER BY lq_score DESC;

-- 3. Park efficiency ranking
SELECT park_name, street, output_per_ha, spatial_enterprises
FROM ontology.park_full
WHERE output_per_ha > 10
ORDER BY output_per_ha DESC;
```

---

## Key Methods

### Location Quotient (LQ)
```
LQ = (e_ij / E_i) / (E_j / E)
LQ > 1.5 → Highly specialized ★
LQ > 1.0 → Specialized ▲
LQ < 1.0 → Not specialized
```

### Cluster Health Score
```
Health = Leadership(weighted by top-firm output) 
       × Density(enterprises per km²) 
       × Innovation(SRDI + high-tech count)
```

### Spatial Band Analysis
```
Core Band (≤500m from arterial roads) → SME incubator zone
Transition Band (500m-2km) → Optimal: highest avg output
Outer Band (>2km) → Large enterprises with independent campuses
```

---

## Architecture

```
L1: Raw Data → Enterprise DB, GIS, Innovation Carriers
L2: Cleaned Views → 25 Materialized Views (ontology.*)
L3: Analysis Engine → 5 Domains × SQL Templates
L4: Report Generator → Markdown reports with embedded insights
```

---

## Who This Is For

- **Government economists** analyzing regional industrial structure
- **Industrial park planners** optimizing land allocation
- **Investment promotion agencies** targeting high-value sectors
- **Data analysts** working with enterprise-level spatial data

---

## License

MIT. Built by practitioners who've done this for real government clients.

<p align="center">
  <sub>5 domains · 25 views · 32 research themes · 1 methodology</sub>
</p>
