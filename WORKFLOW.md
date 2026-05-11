# Workflow: Building a Pre-Computed Gene–Disease GWAS Web App

> **Reusable recipe.** Apply this end-to-end to any "compute one stat per (gene, disease) pair → ship as a Gradio Space" project — drug-target tools, biomarker browsers, phenotype atlases, organ-specific perturbation maps, etc.

This is exactly the workflow that built the [Virtual KO/OE Disease Atlas](./ATLAS.md) (live at https://is.gd/virtual_gene_atlas), distilled into reusable steps with concrete deliverables and gotchas.

---

## 0. Architecture decision (5 min, do this first)

Make these choices up front — they shape every later step.

| Question | Recommendation | Why |
|----------|----------------|-----|
| **Compute mode** | Pre-computed batch, not on-demand | 7M+ pairs run faster offline; users search instantly |
| **Result format** | One parquet file per "primary axis" (disease) | Fast lookup, easy to add/remove, plays well with Git LFS |
| **Figures** | Generate on-demand from parquet, not pre-rendered | Lets you highlight any user-selected gene, way less disk |
| **Web UI** | Gradio on Hugging Face Spaces | Free, simple file upload, zero dev-ops |
| **Storage** | ~1 GB → Git LFS in the Space. Larger → S3 or HF Datasets | LFS handles 1 GB cleanly; beyond that, externalize |

---

## 1. Project layout

```
my-project/
├── app.py                              # Gradio search UI
├── scripts/
│   ├── download_gene_list.py           # HGNC → all approved gene symbols
│   ├── download_gwas_catalog.py        # GWAS Catalog → studies + categories
│   ├── filter_diseases.py              # Curate manageable subset (~hundreds)
│   ├── batch_compute_all.py            # *** core: one pair at a time ***
│   ├── figures.py                      # Shared plotting (ggplot2-style)
│   ├── run_mr_pipeline.py              # Optional: ad-hoc single-disease CLI
│   └── smoke_test.py                   # Pre-deploy sanity check
├── data/
│   ├── raw/
│   │   ├── gene_list.csv               # All HGNC genes (~45k)
│   │   ├── gene_list_protein_coding.csv  # Subset used in batch (~19k)
│   │   ├── disease_list.csv            # Full GWAS Catalog (~90k studies)
│   │   └── disease_list_filtered.csv   # Working set (~hundreds)
│   └── processed/
│       └── disease_<gwas_id>_results.parquet   # One file per disease
├── README.md                           # HF Spaces frontmatter + intro
├── DEPLOY.md                           # HF Spaces upload guide
├── ATLAS.md (or your name)             # Public-facing app description
├── WORKFLOW.md                         # This file
├── requirements.txt
├── .gitignore                          # Whitelist data/processed/*.parquet
└── .gitattributes                      # LFS-track *.parquet
```

---

## 2. Data ingestion (Day 1)

### 2.1 Gene list (HGNC, ~17 MB)
Use the HGNC complete set — single canonical source for human gene symbols + Ensembl/Entrez IDs.

```bash
python scripts/download_gene_list.py
# → data/raw/gene_list.csv (44,986 approved symbols)
# → data/raw/gene_list_protein_coding.csv (19,296 — use this for MR)
```

R equivalent for users who run `biomaRt` daily:
```r
mart <- useMart("ensembl", dataset = "hsapiens_gene_ensembl")
getBM(c("hgnc_symbol", "ensembl_gene_id", "entrezgene_id",
        "chromosome_name", "gene_biotype"), mart = mart)
```

### 2.2 Disease list (GWAS Catalog, ~60 MB)
The EBI GWAS Catalog has two free downloadable files that together give you everything: studies + EFO category mappings.

```bash
python scripts/download_gwas_catalog.py
# → data/raw/disease_list.csv (89,981 studies, 18 categories)
```

Endpoints used (no auth needed):
- `https://www.ebi.ac.uk/gwas/api/search/downloads/studies_alternative`
- `https://www.ebi.ac.uk/gwas/api/search/downloads/trait_mappings`

### 2.3 Curate to a manageable subset
```bash
python scripts/filter_diseases.py --max-per-category 50
# → data/raw/disease_list_filtered.csv (~374 diseases)
```

Filter logic to copy: keep only real disease categories (Cancer, Cardiovascular, Neurological, Immune, Metabolic, …), drop pure "measurements"/biomarkers unless you specifically want them. Cap N-per-category so the batch finishes in hours, not weeks.

**Gotcha:** the full GWAS Catalog has 90k+ studies but most are quantitative traits — your "diseases" list should usually be < 1,000.

---

## 3. Batch compute (Day 1–2)

### 3.1 The core loop
`batch_compute_all.py` follows a single template you can copy verbatim:

```python
for gwas_id in diseases_list:
    parquet_path = outdir / f"disease_{gwas_id}_results.parquet"

    # Resume: skip pairs already in this parquet
    existing = read_parquet(parquet_path) if parquet_path.exists() else empty
    done_genes = set(existing.gene_symbol)
    todo = [g for g in genes_list if g not in done_genes]

    rows = []
    for gene in todo:
        rows.append(compute_one_pair(gene, gwas_id))  # ← your MR call
        log_progress_every(25)

    write_parquet(parquet_path, concat(existing, new_rows))
```

### 3.2 Three properties that matter
1. **Resume-safe.** Long batches get interrupted (laptop sleeps, kernel panics, …). The "skip done genes" pattern lets you restart with `--force` for full re-compute or no flag for resume.
2. **One parquet per disease, not one giant file.** Easier to update (re-run only the diseases that change), faster to query (load only the parquet you need), survives partial corruption.
3. **Deterministic mock first, then real MR.** Get the architecture flowing with random numbers, then swap `compute_one_pair()` for a real `TwoSampleMR::mr()` call. We never had to change any surrounding code.

### 3.3 Performance targets
- **Mock data:** ~2,000 pair/sec → 7M pairs in ~57 min
- **Real OpenGWAS calls:** ~10 pair/sec → would take days
- **Real with locally-downloaded GWAS sumstats:** ~500 pair/sec — this is the production path

Download the GWAS summary statistics ONCE per disease, then loop genes against it locally. Don't hammer the OpenGWAS API.

---

## 4. Figure design (Day 2)

### 4.1 Build a shared `figures.py` module
Both the batch pipeline and the live app generate figures — share the code so they always look the same.

```
scripts/figures.py
├── make_volcano(df, title, ..., highlight=...)   ← returns a Figure
├── make_sim_plot(df, title, ..., highlight=...)
├── save_volcano(path, df, title, ...)            ← wraps + writes PNG
├── save_sim_plot(path, df, title, ...)
└── load_disease_names(...)                       ← gwas_id → readable name
```

### 4.2 ggplot2-style theme in matplotlib
What survives publication scrutiny:
- White background, no grid, only bottom + left spines (`theme_classic()`)
- Color palette: `#D62728` red / `#228B22` forestgreen / `#999999` null
- `adjustText` for ggrepel-like label repulsion (`pip install adjustText`)
- **Color-coded labels** matching point colors — labels stay readable even when packed
- Two stacked legends (Effect + PVE size) using `fig.legend()` (not `ax.legend()`, which truncates text under `bbox_inches='tight'`)

### 4.3 Interpretation polish
- Label legend items with **biological meaning**, not raw stats: "Risk (b > 0)" instead of "b > 0"
- Annotate threshold lines: "P = 0.05" and "Bonferroni P = 3e-06" in the right margin
- Faceted panels for KD vs OE with light-grey strip headers (`facet_wrap`-style)
- Highlight ring (`#FFB300` double ring) on the user-queried gene — make it impossible to miss

### 4.4 Gotchas
- **`bbox_inches='tight'` + `adjust_text`** = stray labels pushed off-axes get captured, blowing up the saved canvas to 5× height. Use a **fixed canvas** with `pad_inches=0.3` and call `ax.set_autoscale_on(False)` before `adjust_text`.
- **`gr.HTML('<script>…</script>')`** silently does nothing — innerHTML doesn't execute scripts per HTML5 spec. Use `demo.load(js=...)` instead.

---

## 5. Search interface (Day 2–3)

### 5.1 Components in `app.py`
Map every user need to a Gradio component:

| User need | Component |
|-----------|-----------|
| Pick a disease | `gr.Dropdown(filterable=True)` |
| Filter by category | `gr.Dropdown` with `.change()` repopulating disease list |
| Single gene | `gr.Dropdown(allow_custom_value=True)` |
| Paste multiple genes | `gr.Textbox(lines=4)` |
| Upload gene list CSV | `gr.File(file_types=['.csv', '.tsv', '.txt'])` |
| Set context size | `gr.Slider` |
| Display results | `gr.Image` + `gr.File` + `gr.Markdown` in tabs |
| Show study metadata | `gr.Markdown` populated on `disease_dd.change()` |

### 5.2 Input priority pattern
When you support multiple input methods, decide ONE priority order and document it:

```python
if uploaded_file: requested = parse(uploaded_file)
elif pasted_text: requested = parse(pasted_text)
elif single_pick: requested = [single_pick]
else: return error
```

Print which mode was used in the status card so users aren't confused.

### 5.3 Disease metadata extraction
The GWAS Catalog `initial_sample_size` field is messy text like:
```
"76,192 European ancestry cases, 63,082 European ancestry controls"
```
A short regex parses it into structured ancestry + cases/controls counts that you can put in a tidy table:

```python
_ANCESTRY_RE = re.compile(
    r"([\d,]+)\s+([A-Z][\w\s/.\-]+?)\s+ancestry"
    r"(?:\s+(cases?|controls?|individuals?|male|female))?",
    re.IGNORECASE,
)
```

---

## 6. Deploy to Hugging Face Spaces (Day 3)

### 6.1 Pre-flight checklist
- [ ] `.gitignore` whitelists `data/processed/*.parquet` and the curated CSVs
- [ ] `.gitattributes` LFS-tracks `data/processed/*.parquet`
- [ ] `README.md` has Spaces frontmatter (`title`, `sdk: gradio`, `sdk_version`, `app_file: app.py`, `short_description` ≤ 60 chars)
- [ ] No secret tokens in code — use `os.getenv('OPENGWAS_JWT')` etc.
- [ ] `requirements.txt` pins major versions (Gradio 6 has breaking API changes vs 4.x)
- [ ] Smoke test passes locally on N diseases × N genes

### 6.2 Push
```bash
hf auth login                                       # paste WRITE token
hf repo create <name> --repo-type space --space_sdk gradio --exist-ok
git lfs install
git lfs track "data/processed/*.parquet"
git add -A && git commit -m "Initial deployment"
git remote add space https://huggingface.co/spaces/<user>/<name>
git push space master:main                          # HF uses 'main'
```

### 6.3 Common push failures (we hit all three)
1. **Push rejected — remote contains work.** HF auto-creates a README on Space init.
   Fix: `git fetch space main && git merge space/main --allow-unrelated-histories -X ours`
2. **Push rejected — file larger than 10 MiB.** Some log/output file slipped in.
   Fix: `git filter-branch --index-filter "git rm -rf --cached --ignore-unmatch logs/" -- --all` then force-push to a brand-new Space (safe — no shared history).
3. **Push rejected — README validation.** `short_description` must be ≤ 60 chars.
   Fix: shorten and re-push.

### 6.4 Verify
- Space page shows "RUNNING"
- `curl https://<user>-<name>.hf.space/` returns Gradio HTML
- App opens, all dropdowns populate, a search returns figures + CSV

---

## 7. Polish for sharing (Day 3)

### 7.1 Hide HF chrome
The bare `.hf.space/` URL shows a floating Space identity card. To hide it for visitors:
1. Append `?embed=true` to the URL — official HF embed mode
2. Add a `demo.load(js=...)` hook that calls `window.location.replace('?embed=true')` on first visit so any URL ends up clean
3. Add a defensive MutationObserver that hides any DOM element with text matching `<owner>/<space-name>` for stubborn cases

### 7.2 Short URL
- **is.gd** with custom alias (no signup, free): https://is.gd → tick "Provide your own short URL"
- TinyURL custom aliases are paywalled in 2026; use is.gd instead

### 7.3 Plot captions
Each result tab should have a 4–6 line **"How to read this plot"** card right below the figure. Non-specialists need this; specialists skim past.

---

## 8. Time budget

| Phase | Realistic time |
|-------|---------------|
| Data ingestion (sections 1–2) | 2–4 hours |
| Batch script + first 100-pair test | 2 hours |
| Full batch run (mock or real) | 1–8 hours **wall**, no babysitting |
| Gradio UI + figures | 4–8 hours |
| HF deploy + chrome polish | 1–2 hours |
| Docs, smoke test, sharing prep | 1–2 hours |
| **Total** | **2–4 working days** |

---

## 9. What to adapt for a different app

The recipe is general. To rebuild for a different topic:

| Layer | What to swap |
|-------|--------------|
| **Primary axis** (was: disease) | Tissue, cell type, drug, age bucket — anything that names a parquet file |
| **Secondary axis** (was: gene) | Variant, pathway, protein, metabolite — anything you loop over inside each parquet |
| **Stats computed** (was: MR β/p + KO/OE sim) | Differential expression, GWAS lookups, pathway enrichment, Cox-regression hazards, … |
| **Highlight semantics** (was: orange ring on queried gene) | Same idea — your "selected entity" gets visually marked among context |
| **Result CSV columns** | Whatever stats your pipeline produces |
| **Threshold lines & legend** | Whatever significance convention applies in your field |
| **Disease-metadata panel** | Swap to relevant per-axis metadata (e.g. tissue cell counts, drug ATC class) |

The architecture (one parquet per primary, on-demand figures, three input modes, search-with-highlight) stays identical.

---

## 10. Reusable code snippets

### One-pair compute template (real GWAS)
```python
def compute_one_pair(gene: str, gwas_id: str) -> dict:
    # 1) Get instrument SNPs for this gene (eQTL or expression-related)
    snps = get_gene_instruments(gene)              # your IV source
    if len(snps) < 2: return null_result()

    # 2) Pull outcome stats from local GWAS sumstats file
    outcome = read_outcome_sumstats(gwas_id, snps)
    if outcome.empty: return null_result()

    # 3) Harmonize + MR
    harm = harmonise(exposure_for(gene, snps), outcome)
    mr = inverse_variance_weighted(harm)

    # 4) Simulate KO/OE
    kd = simulate_perturbation(mr.beta, mr.se, frac=-0.5)
    oe = simulate_perturbation(mr.beta, mr.se, frac=+0.5)

    return {**mr, **kd, **oe, "priority_score": score(mr)}
```

### Highlight a queried entity (universal)
```python
hl_set = {x.upper() for x in (highlight or [])}
hl = d[d["exposure"].str.upper().isin(hl_set)]
ax.scatter(hl.x, hl.y, s=hl["size"] + 250,
           facecolors="none", edgecolors="#FFB300",
           linewidths=3.0, zorder=10)
ax.scatter(hl.x, hl.y, s=hl["size"] + 450,
           facecolors="none", edgecolors="#FFB300",
           linewidths=1.2, alpha=0.5, zorder=9)
```

### HF Spaces frontmatter
```yaml
---
title: <App Name>
emoji: 🧬
colorFrom: red
colorTo: green
sdk: gradio
sdk_version: 6.14.0
app_file: app.py
pinned: false
license: apache-2.0
short_description: <≤60 chars marketing line>
---
```

### Embed-mode auto-redirect (inside `demo.load(js=...)`)
```javascript
() => {
  try {
    var url = new URL(window.location.href);
    if (!url.searchParams.has('embed')) {
      url.searchParams.set('embed', 'true');
      window.location.replace(url.toString());
    }
  } catch (e) {}
}
```

---

## 11. Reference apps

The first app built with this exact workflow:
- **Virtual KO/OE Disease Atlas** — https://is.gd/virtual_gene_atlas
- Source: https://huggingface.co/spaces/jianlizhao/GeneDisease-MR-Browser
- See `ATLAS.md` for the public description.

---

Last updated: 2026-05-11
