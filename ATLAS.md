# 🧬 Virtual KO/OE Disease Atlas

**AI-powered causal simulation of human gene perturbation across diseases**

🔗 **Live app:** https://is.gd/virtual_gene_atlas
*(or the direct URL: https://jianlizhao-genedisease-mr-browser.hf.space/?embed=true)*

---

## What it is

A web tool that lets anyone explore the **causal effect of human genes on disease**, plus simulate what would happen if you knocked down (KO) or overexpressed (OE) each gene — all from pre-computed Mendelian Randomization (MR) results.

- **19,296** protein-coding human genes (HGNC)
- **374** GWAS Catalog diseases (curated from 89,981 studies)
- **7,216,704** pre-computed gene × disease MR results
- **Volcano + KD/OE simulation** figures generated instantly, on demand
- **No login. No install. Works on phone or desktop.**

## Who it's for

- **Drug-target researchers** — find genes whose modulation might lower disease risk
- **GWAS-curious clinicians** — see which genes pop in your disease of interest, with study population context
- **Bioinformatics teams** — prototype hypotheses before committing wet-lab cost
- **Educators** — concrete classroom example of MR causal inference + in-silico perturbation

## How to use it (30 seconds)

1. Open https://is.gd/virtual_gene_atlas
2. **Pick a disease** — filterable dropdown (type "alzheimer", "heart failure", ...)
3. **Tell it which gene(s)** — three options:
   - Single gene: type the HGNC symbol (e.g. `APOE`)
   - Paste a list: `BRCA1, BRCA2, TP53, APOE, IL6` (any separator works)
   - Upload a CSV/TSV/TXT with a gene column
4. **Click Run** → instant volcano + KD/OE simulation + result CSV download

Your chosen genes get a **bright orange ring** on the volcano so they're easy to spot among the top hits.

## Plot interpretation cheat-sheet

### MR volcano (causal direction)
- 🟢 **Green** = **protective effect** (more gene expression → lower disease risk)
- 🔴 **Red** = **risk effect** (more gene expression → higher disease risk)
- **Dotted line** = `P = 0.05` (nominal significance)
- **Dashed line** = Bonferroni-corrected threshold for the # of genes tested
- **Point size** = PVE (proportion of variance explained) — bigger = stronger instrument
- 🟠 **Orange ring** = the gene(s) you queried

### Simulated KD / OE (two panels)
- **Knockdown** (left) = what happens if you reduce gene expression
- **Overexpression** (right) = what happens if you increase it
- Same color code: green = lowers disease risk, red = raises it
- Shared y-axis (`−log₁₀ p`) for easy comparison

### Disease metadata panel (top-left)
When you pick a disease, the app auto-populates:
- Disease name, EFO trait, GWAS Catalog category
- Total sample size + ancestry breakdown
- Cases / controls split
- PubMed link, GWAS Catalog link, OpenGWAS link

## Result CSV columns

| Column | Meaning |
|--------|---------|
| `gene_symbol` | HGNC symbol |
| `gwas_id` | OpenGWAS / GWAS Catalog ID |
| `gwas_beta`, `gwas_se`, `gwas_pval` | GWAS effect size, SE, p-value |
| `gwas_category` | 1 = genome-wide (p < 5e-8), 2 = suggestive, 3 = weak, 4 = none |
| `MR_beta`, `MR_se`, `MR_pval`, `MR_nsnp` | Mendelian Randomization causal estimate |
| `KO_beta`, `KO_pval` | Simulated knockdown effect |
| `OE_beta`, `OE_pval` | Simulated overexpression effect |
| `priority_score` | 0–1 composite (effect size × significance × consistency) |

## Privacy & uptime

- Free public Gradio Space on Hugging Face Spaces (CPU-basic tier)
- **No data submitted by users is stored** — every search runs from pre-computed parquet
- Sleeps after ~48 h idle; first visitor after sleep waits ~30 s for it to wake
- No accounts, no tracking beyond is.gd click counts (anonymous)

## Citation

If you use it in published work, please cite:
- **TwoSampleMR**: Bowden et al. (2015). *Eur J Epidemiol*. doi:10.1007/s10654-015-0011-z
- **OpenGWAS**: Elsworth et al. (2020). *Genome Med*. doi:10.1186/s13073-020-0714-9
- **GWAS Catalog**: Sollis et al. (2023). *Nucleic Acids Research*. doi:10.1093/nar/gkac1010

---

**One-line elevator pitch:**
*Pre-computed MR for every human gene × every major GWAS, with on-demand volcano + KO/OE simulation — for free, no login.*
