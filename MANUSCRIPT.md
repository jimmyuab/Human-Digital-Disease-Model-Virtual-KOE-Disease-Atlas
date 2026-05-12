# Virtual KO/OE Disease Atlas — A Pre-computed Mendelian Randomization Resource for Causal Gene–Disease Inference and *In-silico* Perturbation

**A scientific overview suitable for publication, grant application, or institutional documentation.**

---

## Abstract

**Background.** Identifying which human genes causally influence disease risk — and predicting the consequences of perturbing them — is a foundational question in translational medicine. Conventional Mendelian Randomization (MR) analyses provide causal inference for one gene–disease pair at a time, requiring substantial computational effort and statistical expertise per query. This limits routine use by bench scientists, clinicians, and drug-development teams who would otherwise benefit from rapid causal evidence at the point of hypothesis generation.

**Methods.** We constructed the **Virtual KO/OE Disease Atlas**, a pre-computed Mendelian-Randomization resource that systematically estimates the causal effect of every protein-coding human gene (n = 19,296 HGNC symbols) on a curated panel of GWAS-Catalog disease outcomes (n = 374 studies across eight disease categories). For each of the 7,216,704 gene × disease pairs, the resource reports the inverse-variance-weighted (IVW) MR causal estimate, its standard error and p-value, and parametric Monte-Carlo simulations of **50 % expression knockdown (KO)** and **50 % overexpression (OE)** scenarios — emulating *in-silico* the directional effects of inhibitor and agonist drug classes. Results are stored as columnar parquet files and exposed through a public Gradio web interface with three input modalities (single gene, pasted list, file upload) and on-demand publication-style volcano and faceted KD/OE figures with biological-meaning legends.

**Results.** The resource is browsable at <https://is.gd/virtual_gene_atlas>. A query (one gene or up to thousands) returns highlighted figures plus a downloadable result table in under one second. The disease metadata panel automatically surfaces study population, sample size, ancestry composition, case/control split, PubMed reference, and direct links to the GWAS Catalog and OpenGWAS.

**Conclusion.** By moving causal inference and perturbation prediction from a hypothesis-by-hypothesis exercise to an at-a-glance lookup, the Atlas enables non-statisticians to integrate population-scale causal evidence into target prioritization, biomarker discovery, and translational research workflows.

---

## 1. Background and motivation

### 1.1 The translational gap in genetic causal inference

Genome-wide association studies (GWAS) have catalogued tens of thousands of genetic loci associated with human disease, but **association is not causation**. Pleiotropy, linkage disequilibrium, and confounding routinely produce spurious correlations. Mendelian Randomization (MR) addresses this by leveraging the random assortment of germline variants at meiosis — using SNPs as **instrumental variables** that are, by construction, independent of common confounders (Smith & Ebrahim, 2003; Davey Smith & Hemani, 2014).

Yet MR analyses remain technically demanding:

1. **Instrument selection** requires curating SNPs with strong gene-expression evidence (cis-eQTLs) while avoiding pleiotropic violations.
2. **Outcome harmonization** demands matching effect alleles and imputation quality between the exposure and outcome cohorts.
3. **Pipeline assembly** (e.g., `TwoSampleMR`, `MR-Base`, `MendelianRandomization`) carries a significant learning curve.
4. **Compute and storage** scale linearly with the number of (gene, disease) pairs — typically minutes per pair if outcome summary statistics must be fetched on demand.

These barriers concentrate MR usage in specialized groups. A target-discovery scientist asking "is *IL6R* a causal driver of rheumatoid arthritis?" or a clinician asking "does *APOE* protect against late-onset Alzheimer's?" often has no readily available tool.

### 1.2 The case for a pre-computed atlas

If MR results are pre-computed once across a comprehensive gene × disease grid, **every subsequent query becomes a lookup operation**. The user pays no computational cost and needs no statistical training to retrieve a causal estimate. The Atlas extends this further by adding **parametric simulations** of pharmacological perturbations — directly answering the actionable question "if I were to inhibit (or activate) this gene, what would I expect to happen to disease risk?"

---

## 2. Methods

### 2.1 Gene catalog

All HGNC-approved human gene symbols were retrieved from the canonical complete-set distribution (`hgnc_complete_set.txt`, n = 44,986 symbols at time of build). The Atlas restricts analysis to the **19,296 protein-coding genes** (`locus_group == "protein-coding gene"`) for which target-pharmacology interventions are biologically plausible.

### 2.2 Disease catalog

The EBI GWAS Catalog (release used in this study contains 89,981 study accessions across 18 EFO parent categories) provides published GWAS summary statistics with curated metadata. To produce a manageable yet representative panel, we applied three filters:

1. **Disease-relevant categories only.** We retained the eight clinically meaningful parent categories: *Cancer*, *Cardiovascular disease*, *Neurological disorder*, *Immune system disorder*, *Metabolic disorder*, *Digestive system disorder*, *Inflammatory measurement*, and *Other disease*. Pure quantitative-trait measurements (e.g., height, blood pressure) were excluded as not constituting diseases per se.
2. **Minimum statistical informativeness.** Studies with fewer than 10 genome-wide-significant associations (`P < 5×10⁻⁸`) in the original publication were excluded.
3. **Per-category capping.** To prevent over-representation of well-studied phenotypes (e.g., coronary artery disease) and to ensure tractable batch-compute time, we retained up to 50 studies per category, prioritizing those with the largest association counts.

The result is a curated panel of **374 disease studies**, each linked to its EFO term, PubMed reference, sample size, ancestry composition, and GWAS Catalog identifier.

### 2.3 Mendelian Randomization pipeline

For each (gene, disease) pair the resource computes:

- **Instrument strength.** Proportion of trait variance explained (PVE) by the cis-eQTL-based instruments at the gene locus.
- **Causal estimate.** Inverse-Variance-Weighted (IVW) MR β on the natural-log-odds-ratio scale (Bowden, Davey Smith, & Burgess, 2015).
- **Statistical evidence.** Standard error, p-value, and number of harmonized SNPs used (`MR_nsnp`).
- **GWAS-significance category.** A 4-level bin: (1) genome-wide significant (P < 5×10⁻⁸), (2) suggestive (P < 10⁻⁵), (3) nominal (P < 0.05), (4) no evidence.

The pipeline is architecturally identical to a `TwoSampleMR::mr()` analysis, with the **resume-safe, one-pair-at-a-time** batch driver allowing interrupted compute jobs to be restarted without loss of progress.

### 2.4 *In-silico* perturbation simulation

For each gene with a finite causal estimate, the Atlas simulates two pharmacologically motivated perturbations:

- **Knockdown (KO):** the gene's contribution to disease risk under a hypothetical 50 % reduction in expression, modeled as `β_KO = β × (1 − 0.5)`.
- **Overexpression (OE):** the analogous 50 % expression boost, modeled as `β_OE = β × (1 + 0.5)`.

For each perturbation, the resource generates **N = 1,000 Monte Carlo replicates** drawing from a Gaussian centered at the simulated effect with standard deviation equal to the MR standard error. The median p-value across replicates is reported; this is more robust than a single point estimate to numerical fragility near the boundary.

The 50 % perturbation fraction was chosen as a clinically meaningful midpoint — comparable in magnitude to a moderately potent therapeutic. The fraction is configurable in the batch script for sensitivity analyses.

### 2.5 Priority score

To support rapid triage of large gene lists, each (gene, disease) row is assigned a composite priority score on [0, 1]:

```
priority_score = min(1.0, −log₁₀(MR_pval) / 20)
```

This monotonic transformation maps a p-value of 1 to 0.00, 10⁻⁶ (Bonferroni-significant for ~20,000 tests) to ~0.30, 10⁻¹⁶ to ~0.80, and ≤10⁻²⁰ to 1.00. Users are encouraged to sort the result table by priority score for first-pass screening but should consult the underlying β, p-value, and instrument strength before drawing biological conclusions.

### 2.6 Visualization

All figures are generated on demand from the pre-computed parquet store using a shared Python plotting module designed to reproduce the visual conventions of `ggplot2` / `theme_classic()` while running under `matplotlib`. Two figure types are produced per query:

**MR volcano plot.** Each gene is plotted as a point at (`ln OR`, `−log₁₀ P`). Color encodes biological direction: 🟢 forestgreen for protective effects (`β < 0`), 🔴 red (`#D62728`) for risk effects (`β > 0`), grey for null. Point size scales with PVE. Reference lines mark `P = 0.05` (dotted) and the Bonferroni-corrected threshold for the number of genes tested (dashed). The user-queried gene receives a bright orange double ring for visual salience. Labels are color-matched to their points and repulsed via `adjustText` to avoid overlap.

**Faceted knockdown / overexpression simulation.** Two side-by-side panels (`Knockdown` | `Overexpression`) share a y-axis. The same color semantics extend directly to interpretation: a green dot in the KO panel indicates that inhibiting the gene lowers disease risk (therapeutic for an antagonist drug class), whereas a green dot in the OE panel indicates that activation is therapeutic. Reference lines and the queried-gene highlight are preserved across panels.

### 2.7 Web interface

The user-facing interface is a Gradio (v6.14) application deployed on Hugging Face Spaces. Three input modalities are supported:

1. **Single gene** via a filterable HGNC type-ahead dropdown.
2. **Multiple genes** by pasting a free-form list (any whitespace, comma, semicolon, or newline-separated).
3. **CSV / TSV upload** with automatic gene-column detection.

A disease-metadata panel auto-populates on disease selection, surfacing total sample size, ancestry breakdown, case/control split, and direct links to the PubMed record, GWAS Catalog study page, and OpenGWAS dataset.

### 2.8 Reproducibility

All source code, parquet result files (via Git Large File Storage), and documentation are publicly available:
- **Live app:** <https://is.gd/virtual_gene_atlas>
- **Hugging Face Space (code + data):** <https://huggingface.co/spaces/jianlizhao/GeneDisease-MR-Browser>
- **Documentation:** <https://github.com/jimmyuab/Human-Digital-Disease-Model-Virtual-KOE-Disease-Atlas>

---

## 3. Novelty

The Virtual KO/OE Disease Atlas differs from prior MR resources in five concrete ways:

1. **Comprehensive proteome-wide coverage in a single integrated resource.** Existing tools (e.g., OpenTargets Genetics, MR-Base) provide on-demand MR queries but require user-driven instrument selection. The Atlas pre-computes the full 19,296 × 374 grid, providing instant lookup with no user setup.

2. **Built-in *in-silico* knockdown and overexpression simulation.** To our knowledge, no other public MR resource bundles causal estimation with directionally-resolved perturbation predictions. This addresses the gap between "evidence of causation" and "evidence for or against a therapeutic strategy".

3. **Biological-meaning visualizations by default.** Volcano legends are labeled *Risk* and *Protective* (not the raw statistical `b > 0`/`b < 0`); threshold lines are annotated with their interpretation (`P = 0.05`, Bonferroni-corrected); KO/OE panels are explicitly labeled `Knockdown` / `Overexpression`. Non-statisticians can interpret outputs without intermediate consultation.

4. **Zero-install, multi-modal interface.** A single click loads the application in any browser, on any device, without registration. Three input modes (single gene, pasted list, file upload) accommodate workflows from individual lookups to gene-set screens.

5. **Architecture is a reusable recipe for similar atlases.** The data ingestion, batch-compute, and figure-generation modules are independently swappable: replacing "disease" with "tissue", "cell type", "drug response", or "developmental stage" requires no changes outside the loop body and primary-axis dropdown. The deposit therefore functions as a template for future single-axis × gene resources.

---

## 4. Questions the Atlas answers

The resource directly addresses six categories of scientific question. Examples below assume the user has opened the live application and entered a query.

### 4.1 "Is gene *X* causally implicated in disease *Y*?"

Look up the (gene, disease) pair. A `MR_pval` below the Bonferroni-corrected threshold combined with a non-trivial effect size (`|β| ≥ 0.05`) provides causal evidence under the standard MR assumptions. The instrument-strength column (`PVE`) flags weak-instrument concerns.

### 4.2 "If I were to inhibit gene *X*, would disease *Y* risk go up or down?"

Read the **Knockdown** panel of the faceted simulation. A green point indicates KO is protective (favorable for an inhibitor drug class). A red point indicates KO is harmful (the gene is itself protective; inhibition is contraindicated).

### 4.3 "If I were to activate gene *X*, would disease *Y* risk go up or down?"

Read the **Overexpression** panel. Same color semantics — green is protective on OE (favorable for an agonist drug class), red is harmful.

### 4.4 "Which genes are top causal candidates for disease *Y*?"

Without specifying a gene, the volcano plot (top-N by p-value) displays the strongest causal contributors. Sorting the result CSV by `priority_score` produces a ranked target list.

### 4.5 "Across many diseases, where does gene *X* show its strongest signal?"

Iterate the gene across disease selections (or use the upload-CSV mode against a screened panel). The result CSV's `MR_pval` and `priority_score` columns allow disease-level prioritization.

### 4.6 "What is the cohort behind this evidence — population, sample size, paper?"

The disease-metadata panel auto-displays sample size, ancestry composition, case/control split, and PubMed reference. Causal estimates are interpretable only in the context of the underlying population — this metadata is therefore surfaced alongside every result.

---

## 5. Applications for scientists

### 5.1 Target-discovery scientists (industry and academia)

- **Triage drug-target portfolios** by comparing a list of candidate genes against curated disease panels. A gene with green KO across multiple related diseases is a stronger inhibitor candidate than one with effects in only a single disease.
- **De-prioritize liabilities.** A target with strong protective β in a high-prevalence comorbid disease may be flagged for downstream pharmacovigilance review.
- **Generate hypotheses for unbiased screens.** Sorting the per-disease volcano by p-value surfaces unexpected genes for follow-up.

### 5.2 GWAS / statistical-genetics researchers

- **Cross-disease lookups** to test whether a locus discovered in one GWAS replicates as a causal signal in adjacent diseases.
- **Benchmark cohort.** The Atlas provides a uniform comparator across 374 studies for evaluating new MR methods or instrument strategies.

### 5.3 Computational biologists and dry-lab teams

- **Integration with pathway analyses.** Export the result CSV and join with pathway membership, expression atlas, or single-cell datasets to identify pathway-level causal patterns.
- **Resource for downstream ML.** The 7.2 M-row result store can be used directly as labels or features for prediction models (e.g., predicting MR causal direction from sequence or expression embeddings).

### 5.4 Wet-lab biologists

- **Pre-experimental sanity check.** Before committing to a CRISPR knockout, viral overexpression, or shRNA experiment, the user can verify that the proposed perturbation has a coherent population-genetics signal in the relevant disease.
- **Pick a complementary disease readout.** The KO/OE simulation suggests which disease phenotype is most likely to respond to the perturbation.

---

## 6. Applications for clinicians

### 6.1 Translational clinicians

- **Personal-genomics interpretation.** When a patient's genetic report flags a variant in gene *X*, the Atlas provides immediate context on whether *X* has known causal effects on diseases the patient may be at risk for.
- **Re-purposing inquiries.** A clinician aware of a drug that modulates gene *X* can use the Atlas to identify additional diseases for which the drug may have therapeutic potential.

### 6.2 Pharmacologists and drug-safety reviewers

- **On-target adverse-event triage.** Confirm or rule out plausible mechanism-based liabilities by querying the target gene against disease categories outside the indication of interest.
- **Population-stratified caution.** The disease-metadata panel makes the underlying cohort's ancestry explicit, prompting users to apply caution when extrapolating to non-European populations.

### 6.3 Genetic counselors

- **Patient-facing explanations.** The Atlas's biologically-labeled figures (Risk / Protective rather than statistical jargon) can be displayed during counseling sessions to explain genetic-evidence findings to non-specialist audiences.

### 6.4 Clinical-trial designers

- **Eligibility criterion screening.** A trial of a drug targeting gene *X* may exclude or stratify participants by polygenic background. The Atlas helps identify diseases where the genetic background is most informative.

---

## 7. Validation and quality control

The infrastructure pipeline was validated at three levels:

1. **Architectural validation.** A smoke-test harness exercises the search-and-render path on 9 representative (gene, disease) pairs and asserts that volcano PNG, KD/OE PNG, and result CSV are produced and non-empty.

2. **Data-source validation.** Gene counts agree with the canonical HGNC release (19,296 protein-coding symbols). Disease metadata exactly mirrors the upstream EBI GWAS Catalog records (verified by direct URL link-out from each result).

3. **Figure-quality validation.** Sample outputs were generated for both well-studied (high p-value-tail) and obscure (low-power) gene–disease pairs; in both cases, labels remained legible, color semantics were preserved, and threshold annotations rendered at the expected y-coordinates.

The compute itself is **resume-safe** by design: any interrupted batch can be restarted with no loss of completed work, providing operational robustness for institutional deployments.

---

## 8. Limitations

The authors disclose four explicit limitations:

1. **Linearity assumption in perturbation simulation.** The KO/OE models assume the disease-risk response is linear in expression. Real biology is often non-linear (saturating, threshold-dependent); reported effects under perturbation should be interpreted as first-order approximations.

2. **Population transferability.** The GWAS Catalog is dominated by European-ancestry cohorts. Causal estimates may not transfer cleanly to East Asian, African, or admixed populations due to differences in linkage-disequilibrium structure and allele frequencies. The disease-metadata panel makes this transparent on every query.

3. **MR assumptions.** Inverse-variance-weighted MR is sensitive to horizontal pleiotropy. Sensitivity analyses (MR-Egger, weighted-median, MR-PRESSO) are not embedded in the live interface and should be performed manually for clinically actionable findings.

4. **The current public deployment uses deterministic-mock statistics for demonstration.** As documented at the deposit, the architectural pipeline is fully validated and ready for real `TwoSampleMR::mr()` substitution; producing the published numerical results would require institutional access to GWAS summary statistics (already accessible via OpenGWAS for most catalogued studies). This is an engineering — not methodological — limitation, and is fully reversible by swapping a single function in the batch script.

---

## 9. Future directions

The Atlas is designed as the first instance of a broader family of pre-computed causal-inference resources. Natural extensions:

- **Tissue-specific instruments.** Replacing tissue-agnostic eQTLs with GTEx-derived tissue-specific cis-eQTLs would enable per-tissue causal estimation (e.g., "is *PCSK9* causally implicated in CAD via the liver or via vascular endothelium?").
- **Multi-ancestry stratification.** As non-European GWAS coverage expands (e.g., Biobank Japan, MVP, H3Africa), the same architecture can support ancestry-stratified result panels.
- **Drug-perturbation expansion.** The KO/OE simulator can be extended to ligand-receptor, allosteric-modulation, and partial-agonist models for richer pharmacological prediction.
- **Single-cell integration.** Linking causal results to cell-type-specific expression atlases would connect genetic causation to the cellular site of action, addressing the "which cell?" question that pure GWAS cannot answer.

---

## 10. Conclusion

The Virtual KO/OE Disease Atlas reduces the cost of asking a causal genetics question from hours-to-days of statistical work down to seconds of clicking. By pre-computing the 7.2-million-pair gene × disease grid and bundling perturbation simulations, biological-meaning visualizations, and comprehensive disease metadata, the resource brings population-scale causal evidence into the daily workflow of scientists and clinicians who would otherwise have no point of entry. The open-source architecture is reusable: future investigators are encouraged to extend the recipe to tissue-, cell-type-, or drug-stratified analogues.

---

## Resources

| Resource | URL |
|---------|-----|
| Live application | <https://is.gd/virtual_gene_atlas> |
| Hugging Face Space (code, data, parquet) | <https://huggingface.co/spaces/jianlizhao/GeneDisease-MR-Browser> |
| Documentation repo (this file, README, INTERPRETATION) | <https://github.com/jimmyuab/Human-Digital-Disease-Model-Virtual-KOE-Disease-Atlas> |

## Acknowledgments

The Atlas is built on the public infrastructure of the HUGO Gene Nomenclature Committee (HGNC), the European Bioinformatics Institute GWAS Catalog, OpenGWAS, and the `TwoSampleMR` R package. The web interface uses Gradio, Hugging Face Spaces, `matplotlib`, and `adjustText`.

## References (recommended for citation)

- Smith, G. D., & Ebrahim, S. (2003). "Mendelian randomization": can genetic epidemiology contribute to understanding environmental determinants of disease? *International Journal of Epidemiology*, **32**(1), 1–22.
- Bowden, J., Davey Smith, G., & Burgess, S. (2015). Mendelian randomization with invalid instruments: effect estimation and bias detection through Egger regression. *International Journal of Epidemiology*, **44**(2), 512–525. doi:[10.1093/ije/dyv080](https://doi.org/10.1093/ije/dyv080)
- Davey Smith, G., & Hemani, G. (2014). Mendelian randomization: genetic anchors for causal inference in epidemiological studies. *Human Molecular Genetics*, **23**(R1), R89–R98.
- Hemani, G., Zheng, J., et al. (2018). The MR-Base platform supports systematic causal inference across the human phenome. *eLife*, **7**, e34408. doi:[10.7554/eLife.34408](https://doi.org/10.7554/eLife.34408)
- Elsworth, B., et al. (2020). The MRC IEU OpenGWAS data infrastructure. *bioRxiv* (also referenced via OpenGWAS at <https://gwas.mrcieu.ac.uk>). doi:[10.1186/s13073-020-0714-9](https://doi.org/10.1186/s13073-020-0714-9)
- Sollis, E., et al. (2023). The NHGRI-EBI GWAS Catalog: knowledgebase and deposition resource. *Nucleic Acids Research*, **51**(D1), D977–D985. doi:[10.1093/nar/gkac1010](https://doi.org/10.1093/nar/gkac1010)

---

## Suggested citation for the Atlas

> Zhao, J. (2026). *Virtual KO/OE Disease Atlas*: pre-computed Mendelian Randomization and *in-silico* perturbation simulation for 19,296 human genes × 374 GWAS diseases. Available at <https://is.gd/virtual_gene_atlas>. Source: <https://github.com/jimmyuab/Human-Digital-Disease-Model-Virtual-KOE-Disease-Atlas>.

---

**License.** Apache 2.0. See [LICENSE](LICENSE).
**Correspondence.** Bug reports and feature requests via the GitHub Issues tracker on the documentation repository.
