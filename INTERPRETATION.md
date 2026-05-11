# How to Read the Virtual KO/OE Disease Atlas — Detailed Interpretation Guide

This file explains, element by element, exactly what every plot and every field means and how to draw scientific conclusions from them. Read it once and you can interpret any result the app produces.

> Looking for the **list of available diseases** (all 374, organized by category) or the **gene catalog** (19,296 protein-coding symbols)? See **[DATA.md](DATA.md)**.

---

## Table of contents
1. [The disease metadata panel](#1-the-disease-metadata-panel)
2. [The MR volcano plot — element by element](#2-the-mr-volcano-plot--element-by-element)
3. [The simulated knockdown / overexpression plot](#3-the-simulated-knockdown--overexpression-plot)
4. [The result CSV — every column explained](#4-the-result-csv--every-column-explained)
5. [The priority score](#5-the-priority-score)
6. [Putting it all together — three worked examples](#6-putting-it-all-together--three-worked-examples)
7. [Limitations and caveats](#7-limitations-and-caveats)

---

## 1. The disease metadata panel

The moment you pick a disease, a card appears on the left side of the app:

```
📋 Breast cancer

| GWAS ID                  | ebi-a-GCST004988                                |
| Study accession          | GCST004988  → GWAS Catalog                      |
| Category                 | Cancer                                          |
| EFO trait                | breast carcinoma                                |
| Publication              | PubMed 29059683  → NCBI                         |
| # GWAS-significant assoc | 803                                             |
| Total sample size        | 139,274                                         |
| Ancestry / population    | European (139,274)                              |
| Cases / controls         | case: 76,192 · control: 63,082                  |
| Sample (verbatim)        | 76,192 European cases, 63,082 European controls |

🔗 Open in GWAS Catalog · OpenGWAS
```

Field-by-field meaning:

### 1.1 GWAS ID
The **OpenGWAS** identifier — what you'd use to call the API or look up the underlying summary statistics. Format: `ebi-a-GCSTxxxxxx`. The prefix `ebi-a-` indicates this is a GWAS Catalog-sourced study.

### 1.2 Study accession
The **GWAS Catalog** identifier (without the `ebi-a-` prefix). Clicking the link takes you to the canonical study page with full curated metadata at https://www.ebi.ac.uk/gwas/studies/GCSTxxxxxx — including the GWAS lead variants, effect alleles, the original paper, and quality flags.

### 1.3 Category
The EFO parent term that the study's trait rolls up to. One of:
- **Cancer** — solid tumors, blood malignancies, etc.
- **Cardiovascular disease** — coronary artery disease, heart failure, atrial fibrillation, …
- **Neurological disorder** — Alzheimer's, Parkinson's, MS, epilepsy, …
- **Immune system disorder** — autoimmune, allergies, IBD, lupus, …
- **Metabolic disorder** — T1D, T2D, obesity-related, lipid disorders, …
- **Digestive system disorder** — IBD, ulcerative colitis, Crohn's, GERD, …
- **Inflammatory measurement** — CRP, IL-6, TNF-α biomarkers, …
- **Other disease** — rare conditions, infectious diseases

Use the category filter at the top of the app to narrow the disease dropdown.

### 1.4 EFO trait
The canonical [Experimental Factor Ontology](https://www.ebi.ac.uk/efo/) name — more precise than the verbatim disease name (which can vary by study). For example, multiple studies might all map to the EFO trait `breast carcinoma` even if their study names differ ("Breast cancer", "Breast cancer (HER2+ subtype)", etc.).

### 1.5 Publication (PubMed link)
Direct link to the original paper's abstract on NCBI. **Always check this** — the paper is the authoritative source for the cohort, methods, and interpretation.

### 1.6 # GWAS-significant associations
How many independent SNPs reached genome-wide significance (`P < 5×10⁻⁸`) in the original GWAS. Rough proxy for "how powered was this study":
- **< 10** → low power, likely missing many true signals
- **10–50** → moderate
- **50–200** → well-powered
- **200+** → very large cohort, robust

For *Breast cancer* above: 803 hits is exceptional — this is one of the most-powered cancer GWAS to date.

### 1.7 Total sample size
Sum of cases + controls + any quantitative-trait individuals across all groups in the study. **This is the single most predictive number for whether MR results will be trustworthy.** Bigger is almost always better.

| Range | Power for MR |
|-------|-------------|
| < 5,000 | Very limited — most causal effects undetectable |
| 5,000–50,000 | Moderate — large effects detectable |
| 50,000–500,000 | Good — most moderate effects detectable |
| > 500,000 | Excellent — even small effects detectable |

### 1.8 Ancestry / population
Auto-parsed from the GWAS Catalog `initial_sample_size` text. Tells you which population the causal estimates apply to. Almost all big GWAS are European-ancestry; estimates may not transfer to other populations because:
- Different LD structure → different instrument behavior
- Different allele frequencies
- Different baseline disease prevalence

**If your patient population is non-European and the study is European-only, treat all results as hypothesis-generating, not actionable.**

### 1.9 Cases / controls
Split from the same parse. The case:control ratio matters for power calculations:
- Ideal balanced design ~ 1:1 to 1:4
- Heavy imbalance (1:20+) → effective sample size is approximately `4 × cases × controls / total`

### 1.10 Sample (verbatim)
The raw `initial_sample_size` string from the GWAS Catalog — verbatim, no parsing. Useful when the auto-parse misses nuance (e.g., "12,863 European male cases, 19,521 European female cases…" gets summed but the auto-parse won't separate the sex split).

### 1.11 Footer links
- **GWAS Catalog** — full curated record (study, traits, lead variants, sample composition)
- **OpenGWAS** — summary statistics file + REST API endpoint

---

## 2. The MR volcano plot — element by element

This is the headline plot. Every point = one gene's causal effect on the chosen disease.

### 2.1 The axes

**Y-axis: `−log₁₀(P-value)`**
- A standard transformation that turns "lower p = more significant" into "higher number = more significant" (easier on the eye).
- P = 0.05 → −log₁₀ ≈ **1.3**
- P = 10⁻⁸ → −log₁₀ = **8**
- P = 10⁻¹⁰⁰ → −log₁₀ = **100** (rare in real data; common in our mock-stat demo)
- Points **higher** on the plot = **stronger statistical evidence** that the gene affects the disease.

**X-axis: `ln(OR)` — the natural log of the Odds Ratio (MR causal estimate)**
- `OR = 1.0` → no effect → x = 0 (the **dashed vertical line**)
- `OR > 1` → x positive → exposure raises disease odds → **risk-increasing**
- `OR < 1` → x negative → exposure lowers disease odds → **protective**
- The magnitude indicates effect size:
  - `ln(OR) = 0.1` → OR ≈ 1.1 → 10 % higher odds — small/moderate effect
  - `ln(OR) = 0.7` → OR ≈ 2.0 → twice the odds — large effect
  - `ln(OR) = −0.3` → OR ≈ 0.74 → 26 % lower odds — moderate protective

### 2.2 Point color — biological interpretation

| Color | Meaning | Causal direction |
|-------|---------|------------------|
| 🟢 **Forestgreen** `#228B22` | **Protective** | More gene → less disease risk |
| 🔴 **Red** `#D62728` | **Risk** | More gene → more disease risk |
| ⚪ **Grey** `#999999` | Null / NA / direction undefined | No causal effect or insufficient data |

**Key point:** "more of the gene" in MR usually means "the genetic variants that increase expression". The interpretation is **causal**, not just associational, because MR uses germline variants (randomly assorted at conception) as instruments.

### 2.3 Point size — PVE (Proportion of Variance Explained)

The diameter of each dot scales with how much trait variance the gene's instruments explain. Bigger dot = stronger / more confident instrument.

| Dot size | PVE | Strength |
|----------|-----|----------|
| Tiny | 0.05 | Weak instrument — interpret causal estimate cautiously |
| Small | 0.10 | OK |
| Medium | 0.15 | Strong |
| Big | 0.20+ | Very strong, robust instrument |

Weak instruments (`PVE < 0.05`) can give biased MR estimates if the F-statistic is low; always interpret tiny dots with skepticism even if the p-value looks compelling.

### 2.4 Reference lines

**Vertical dashed line at `x = 0`**
- The boundary between protective and risk effects. Points exactly on this line have no causal effect.

**Horizontal dotted line at `y = −log₁₀(0.05) ≈ 1.30`**
- The **nominal significance threshold** (`P = 0.05`).
- Points **above** this line are nominally significant.
- Annotated `P = 0.05` in the right margin.
- ⚠️ With thousands of genes tested, ~5 % of points will appear above this line by chance — you almost never want this threshold alone.

**Horizontal dashed line at the Bonferroni-corrected threshold**
- `P_Bonferroni = 0.05 / n_genes_tested`
- For a 19,296-gene scan: `P_Bonferroni ≈ 2.6 × 10⁻⁶` → `−log₁₀ ≈ 5.6`.
- Annotated `Bonferroni P = …` in the right margin.
- Points above this line survive multiple-testing correction.
- **This is the threshold to trust for "discovery"** in a hypothesis-free scan.

For typical disease GWAS, points should ideally sit above the Bonferroni line before claiming a finding.

### 2.5 Gene labels

Color-coded text matching the point color (red text = risk gene, green = protective). The exact gene's HGNC symbol is shown. Labels use `adjustText`-style repulsion to spread them out and connect back to their points with thin grey lines.

**The label color tells you the direction at a glance** — even before reading the gene name you know whether it's a risk or protective hit.

### 2.6 The orange double ring 🟠

The gene(s) **you queried** (via dropdown / paste box / file upload) are circled twice in bright orange (`#FFB300`):
- Inner solid ring — definitive marker
- Outer faded ring — wider "halo" for easy spotting from a distance

The ring appears even when your queried gene isn't in the top-50 displayed labels — it's drawn at the actual data point coordinates so you can locate the gene anywhere on the volcano.

If you uploaded 10 genes, you'll see 10 orange rings scattered across the plot, telling you at a glance:
- Which of your gene set are causally strong (rings far from origin, high on y-axis)
- Which are non-significant null hits (rings clustered around `(0, 0)`)
- Which are protective vs risk (left vs right of zero)

### 2.7 The legend (right side)

**Effect legend (top)**
Three markers — red, green, grey — with labels:
- **Risk (b > 0)** — same as positive `ln(OR)`
- **Protective (b < 0)** — same as negative `ln(OR)`
- **Null / NA**

The notation `b` is just the MR coefficient, identical to `ln(OR)`. Both labels are present for transparency.

**PVE size legend (bottom)**
Four reference dots at PVE = 0.05, 0.10, 0.15, 0.20, so you can map any point's size back to a number.

### 2.8 The title and subtitle

Title: **disease name** (e.g. "Breast cancer") — what you're studying.
Subtitle: `GWAS: ebi-a-GCST004988 · category: Cancer · N = 19,296 genes · highlighting 5 of 5 requested · plot shows 35 points`
- The GWAS ID is shown so you can cite the exact study
- N = how many genes were tested in the batch
- Highlighting = how many of your queried genes the app found in this disease's results
- Plot shows = the displayed top-N (default 50) + your queried genes

### 2.9 How to make a scientific call

A real "drug-target candidate" needs **multiple things to align** on the volcano:
1. **Above the Bonferroni line** (`−log₁₀ p > 5.6` for 19k genes)
2. **|`ln(OR)`| ≥ 0.05** at minimum — anything smaller is too small a clinical effect even if statistically clear
3. **Large dot** (`PVE ≥ 0.05`) — strong instrument
4. **Color matches your therapeutic direction** — green if you want to develop an activator/agonist, red if you want an inhibitor/antagonist

A point that satisfies all four is a strong **causal evidence** for a drug target. One that fails any of them is hypothesis-generating, not actionable.

---

## 3. The simulated knockdown / overexpression plot

This is the **in-silico drug development** plot. Two panels side-by-side.

### 3.1 The two panels

**Knockdown (left)**
- Simulates **reducing** the gene's expression (e.g., as a small-molecule inhibitor, siRNA, gene-edit deletion would do).
- Compute: take the MR effect `b` and multiply by `(1 − 0.5) = 0.5` — assuming a 50 % reduction in expression.
- Re-compute the simulated effect on disease risk under this perturbation.

**Overexpression (right)**
- Simulates **increasing** expression (e.g., a small-molecule agonist, gene-therapy delivery, expression construct).
- Compute: take `b` and multiply by `(1 + 0.5) = 1.5` — assuming a 50 % expression boost.

### 3.2 What the colors mean now

The colors **directly tell you the therapeutic outcome** of the perturbation:

| Panel | Color | Interpretation |
|-------|-------|----------------|
| Knockdown | 🟢 Green | KO **lowers** disease risk — gene is a *risk-driver*; inhibition is therapeutic |
| Knockdown | 🔴 Red | KO **raises** disease risk — gene is *protective*; don't inhibit |
| Overexpression | 🟢 Green | OE **lowers** disease risk — gene is *protective*; activation is therapeutic |
| Overexpression | 🔴 Red | OE **raises** disease risk — gene is *risk-driver*; don't activate |

The two panels are mirror images of each other (mathematically: if KO is green, OE is automatically red, and vice versa). They show **the same biology from both intervention angles** — useful because in practice some drug classes only work in one direction.

### 3.3 Axes

- **X-axis:** `Mean simulated ln(OR)` — same scale as the volcano
- **Y-axis:** `−log₁₀(median P)` — uses the **median** p-value across N=1000 Monte Carlo replicates per perturbation. More robust than a single estimate.
- Shared y-axis between the two panels so they're directly comparable

### 3.4 Reference lines

**Vertical dashed line at x = 0** — boundary between effect directions.
**Horizontal dashed line at `P = 0.05`** (`−log₁₀ ≈ 1.3`) — significance threshold. Annotated in the right margin of the Overexpression panel.

No Bonferroni line here — the sim plot already only shows the top-12-per-direction-per-panel labels (24 genes max per panel), so multiple-testing concern is less acute.

### 3.5 The grey strip headers

Each panel has a light grey rectangle at the top with the panel label (`Knockdown`, `Overexpression`). This is the matplotlib equivalent of ggplot2's `facet_wrap` strips. Just labels — no other meaning.

### 3.6 The orange ring

Same as the volcano — circles the gene(s) you queried, but now you'll see them in **both panels** at corresponding positions. If a gene's queried-ring is green in KO and red in OE, that's a **clean, actionable risk-driver** that should be inhibited (target for an antagonist).

### 3.7 How to read it for drug discovery

Look at your queried gene in the KO panel:
- 🟢 **Green + above the threshold line** → strong candidate for inhibition. Develop antagonist / siRNA / antibody. (Examples in real biology: PCSK9 inhibition lowers LDL, IL-6R inhibition treats RA.)
- 🔴 **Red + above the line** → don't inhibit; the gene is protective. Either leave it alone or try to *boost* it instead (look at the OE panel — should be green there).
- ⚪ **Near zero + below the line** → no actionable signal from this analysis. Either the gene doesn't causally affect this disease, or the underlying MR was underpowered.

---

## 4. The result CSV — every column explained

Download the CSV from the "Result CSV" tab. One row per gene you requested. Columns:

| Column | Type | What it means |
|--------|------|---------------|
| `gene_symbol` | str | HGNC official symbol (e.g., `BRCA1`) |
| `gwas_id` | str | OpenGWAS ID of the disease (e.g., `ebi-a-GCST004988`) |
| `gwas_beta` | float | Effect size from the GWAS, on the natural-log-odds-ratio scale |
| `gwas_se` | float | Standard error of `gwas_beta` |
| `gwas_pval` | float | P-value from the GWAS at the lead variant for this gene |
| `gwas_category` | int | Significance bin: 1 = genome-wide (P<5e-8), 2 = suggestive (P<1e-5), 3 = nominal (P<0.05), 4 = none |
| `pve` | float | Proportion of variance explained by the gene's instruments (0.005 to 0.20+) |
| `n_iv_snps` | int | Number of instrument-variable SNPs used |
| `MR_method` | str | MR method used (default: `Inverse variance weighted`) |
| `MR_beta` | float | The MR causal estimate (`ln OR`) — **this is the main number to interpret** |
| `MR_se` | float | Standard error of `MR_beta` |
| `MR_pval` | float | P-value for the MR causal estimate |
| `MR_nsnp` | int | Number of SNPs that survived MR harmonization |
| `KO_beta` | float | Simulated effect size under 50 % knockdown |
| `KO_pval` | float | Median p-value across 1000 KO Monte Carlo replicates |
| `OE_beta` | float | Simulated effect size under 50 % overexpression |
| `OE_pval` | float | Median p-value across 1000 OE replicates |
| `priority_score` | float | 0–1 composite ranking (see [Priority score](#5-the-priority-score)) |
| `figure_dir` | str | Path to the per-pair figure files (used by the app) |

If a gene **wasn't found** in the disease's pre-computed results (e.g., uploaded an unknown symbol), it appears at the bottom of the CSV with:
- `gene_symbol` filled in
- `gwas_id` filled in
- `notes` = `"Not in pre-computed results for this disease"`
- All numeric columns empty

---

## 5. The priority score

A 0–1 composite that combines effect size + significance into a single "look at this gene first" ranking.

Computed as:
```
priority_score = min(1.0, −log₁₀(MR_pval) / 20)
```

In English: "how many orders of magnitude past `P = 1` is the MR p-value, divided by 20 to cap at 1.0."

| Score | Roughly equivalent p-value | Interpretation |
|-------|-----------------------------|---------------|
| 0.00 | ≥ 1 | No signal |
| 0.10 | ~10⁻² | Nominal hit |
| 0.30 | ~10⁻⁶ | Bonferroni-significant for 20k tests |
| 0.50 | ~10⁻¹⁰ | Highly significant |
| 0.80 | ~10⁻¹⁶ | Very robust hit |
| 1.00 | ≤ 10⁻²⁰ | Top-tier evidence |

**Sort the result CSV by `priority_score` descending** to triage which genes to read first.

Note: priority_score does NOT account for biological plausibility, drug tractability, or expression in the relevant tissue. It's a pure-statistics ranking. Always cross-reference with prior literature before acting on it.

---

## 6. Putting it all together — three worked examples

### 6.1 A clean risk-gene candidate (drug-inhibition target)

You query `IL6R` against `Rheumatoid arthritis`. You see:
- 🔴 Red dot **far above** the Bonferroni line, at `ln(OR) ≈ 0.18` (OR ≈ 1.2)
- Big dot (`PVE ≈ 0.12`) → strong instrument
- Disease metadata: 50,000 cases European RA GWAS → well-powered
- KO simulation: 🟢 Green, p < 10⁻⁸ → inhibiting IL6R lowers disease risk
- Priority score: 0.85

**Conclusion:** genetic evidence supports IL6R inhibition for RA. (This is exactly what tocilizumab does in real clinical practice — the MR matches the drug's known efficacy.)

### 6.2 A protective gene (potential activation target)

You query `ABCA1` against `Coronary artery disease`. You see:
- 🟢 Green dot **above the line**, at `ln(OR) ≈ −0.15`
- Small dot (`PVE ≈ 0.04`) → weak-ish instrument, take with caution
- Disease metadata: 100,000+ case CAD GWAS → well-powered
- OE simulation: 🟢 Green, p < 10⁻⁶ → boosting ABCA1 lowers CAD risk
- Priority score: 0.55

**Conclusion:** consistent with ABCA1 being protective (HDL biology), genetic evidence supports activation strategy. The small dot says "MR may be slightly biased — confirm with a different instrument or replication cohort."

### 6.3 No actionable signal

You query `GRK5` against `Aerodigestive squamous cell cancer`. You see:
- Tiny grey dot **right at the origin** `(0, 0)`
- Tiny dot (`PVE ≈ 0.005`) → barely an instrument
- KO/OE both have orange rings hovering at `(0, 0)`
- Priority score: 0.006

**Conclusion:** no causal signal between GRK5 and this cancer. Either the gene doesn't affect this disease, or this GWAS isn't powered enough to detect a modest effect. Move on or try a related disease.

---

## 7. Limitations and caveats

Be aware of these before drawing strong conclusions:

### 7.1 MR assumptions
MR requires three assumptions:
1. **Relevance** — the SNP instrument affects the gene
2. **Independence** — the instrument is independent of confounders
3. **Exclusion restriction** — the instrument affects the disease *only* through the gene

Violations (especially #3, *horizontal pleiotropy*) can give false-positive causal estimates. The app uses Inverse-Variance-Weighted MR, which doesn't formally test for pleiotropy. Sensitivity analyses (MR-Egger, weighted median) aren't shown — verify your top hits in the original literature.

### 7.2 Population
Most of the GWAS catalog is European-ancestry. Causal estimates may not transfer to other ancestries due to LD/allele-frequency differences.

### 7.3 Tissue-specificity
This atlas uses tissue-agnostic instruments. Causal effects might be tissue-specific (e.g., expressing a gene in liver vs brain can have opposite effects). Cross-reference with single-cell / tissue-expression atlases.

### 7.4 The simulation is parametric
KO and OE estimates assume linearity (`b_simulated = b_observed × (1 ± fraction)`). Real biology is often non-linear (saturable, threshold-dependent). The simulated p-values use 1000 Monte Carlo replicates from a Gaussian with mean = `b_simulated` and SD = the MR `se`.

### 7.5 The current pipeline uses deterministic mock statistics
**Important context for the demo deployment:** the public Atlas at https://is.gd/virtual_gene_atlas uses *deterministic-mock* numbers (same gene + same disease always returns the same simulated value) so the architecture can be evaluated end-to-end without a long compute. To use real numbers, swap `compute_one_pair()` in `scripts/batch_compute_all.py` (on the [Hugging Face Space](https://huggingface.co/spaces/jianlizhao/GeneDisease-MR-Browser)) with a real `TwoSampleMR::mr()` call and re-run the batch.

### 7.6 P-values are NOT effect sizes
A `P = 10⁻³⁰⁰` finding can have a tiny `ln(OR) = 0.02` (a 2 % effect — clinically irrelevant). Always look at both axes together:
- **Up + right** = strong risk evidence
- **Up + left** = strong protective evidence
- **Up + near zero** = statistically significant but clinically trivial — be skeptical

---

## Quick-reference flashcard

Print or screenshot:

| Question | Look at |
|----------|---------|
| Is this gene causally linked to the disease? | Y-axis position — above Bonferroni line? |
| Risk or protective? | Color (red = risk, green = protective) |
| How big is the effect? | X-axis distance from zero (`\|ln OR\| ≥ 0.05` for clinical relevance) |
| How strong is the instrument? | Dot size (PVE ≥ 0.05 ideal) |
| Should I inhibit (drug target)? | KO panel: green = yes, red = no |
| Should I activate? | OE panel: green = yes, red = no |
| Whom does this apply to? | Disease panel "Ancestry / population" field |
| Trust the numbers? | Disease panel total sample size; PubMed paper for methods |
| What to read first? | Sort result CSV by `priority_score` descending |

---

Last updated: 2026-05-11
