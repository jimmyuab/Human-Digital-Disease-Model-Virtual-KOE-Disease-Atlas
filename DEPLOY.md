# Deploy to Hugging Face Spaces

Upload the pre-computed results + Gradio app to a public/private HF Space.

## What gets uploaded

| Path | Size | Required |
|------|------|----------|
| `app.py` | 8 KB | ✅ |
| `scripts/figures.py` | 12 KB | ✅ — imported by app |
| `requirements.txt` | <1 KB | ✅ |
| `data/processed/disease_*_results.parquet` | **~906 MB total** (374 files × ~2.4 MB) | ✅ |
| `data/raw/disease_list_filtered.csv` | ~50 KB | ✅ — drives disease dropdown |
| `data/raw/gene_list_protein_coding.csv` | ~1.2 MB | ✅ — drives gene typeahead |
| `data/raw/disease_list.csv` (full 90k) | 17 MB | ⛔ optional, not used by app |
| `data/raw/all_human_genes.tsv` | 17 MB | ⛔ optional, regenerable |
| `data/raw/gwas_catalog_*.tsv` | 60 MB | ⛔ optional, regenerable |
| `docs/entities/disease/*/` aggregate PNGs | ~75 MB | ⛔ optional (app generates on-demand) |

**Minimum upload: ~910 MB.** Comfortably under HF Spaces free-tier 50 GB disk limit; Git LFS is required for parquet files. If you want to shrink the upload: drop the `figure_dir` column from the parquets (it's regenerable from gwas_id + gene_symbol) — saves ~30 %.

## Step 1 — Create the Space

```bash
huggingface-cli login                       # paste your HF write token
huggingface-cli repo create \
  --type space --space-sdk gradio \
  GeneGWAS-MR-Simulator
```

Or via the web: [huggingface.co/new-space](https://huggingface.co/new-space) → SDK = **Gradio** → public/private.

## Step 2 — Track parquets with Git LFS

```bash
cd D:\MR
git lfs install
git lfs track "data/processed/*.parquet"
git add .gitattributes
```

## Step 3 — Tighten .gitignore for the upload

Keep `data/raw/disease_list_filtered.csv` and `gene_list_protein_coding.csv`, but exclude the bigger raw catalog files:

```gitignore
# data/raw/all_human_genes.tsv      # already ignored
# data/raw/gwas_catalog_*.tsv       # already ignored
# data/raw/disease_list.csv         # full catalog — optional
```

(The current `.gitignore` already excludes `data/raw/` broadly. Override per-file with `!path` if needed.)

## Step 4 — Push to the Space

```bash
# Add HF remote
git remote add space https://huggingface.co/spaces/<your-username>/GeneGWAS-MR-Simulator

# Push (auto-triggers Docker build on the Space)
git push space main
```

Building takes 3–8 min: install Python deps, then start `python app.py`. Watch progress in the Space's **Logs** tab.

## Step 5 — (optional) Secrets

The current app uses only pre-computed parquet results — no API calls — so **no Space Secrets are required**. If you later wire up live OpenGWAS queries:

1. Space **Settings → Repository secrets**
2. Add `OPENGWAS_JWT` = `<your_new_token>`
3. Auto-injected as `os.getenv("OPENGWAS_JWT")` in code

## Step 6 — Smoke test

Once Space says **Running**:

1. Open the embedded preview URL.
2. Pick a category → pick a disease → type a gene (e.g. `GRK5`).
3. Click **Run / Show results**.
4. Verify all 3 tabs show artifacts and downloads work.

## Updating after a re-batch

If you re-run `batch_compute_all.py` and want the Space to reflect the latest:

```bash
git add data/processed/*.parquet
git commit -m "Refresh pre-computed results"
git push space main
```

Space will re-deploy on push.

## Scaling further

The current Space ships parquet for 374 filtered diseases. To expand:

1. Re-filter — pick more diseases (e.g. `--max-per-category 200` → ~1,500 diseases, ~870 MB parquet).
2. Re-run batch.
3. Push.
4. If the Space hits the 50 GB free-tier disk limit, consider:
   - Hosting parquet on S3 / HF Datasets and `download_from_hub()` at startup
   - Or upgrading the Space to a paid tier with more disk

## Local dev vs. Space

The app autodetects what's available — works identically locally and on Space:
```bash
python app.py             # → http://localhost:7860
```
