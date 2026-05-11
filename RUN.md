# How to Run the Virtual KO/OE Disease Atlas

Three ways, listed easiest first.

| Path | Audience | What you do |
|------|----------|-------------|
| **A.** Just use the live app | Anyone | Click one link — see below |
| **B.** Run a local copy on your machine | Developer / researcher | Clone the **Hugging Face Space** (it has the code + data), install deps, `python app.py` |
| **C.** Re-deploy your own copy to HF Spaces | Lab owner / collaborator | Fork the Space, edit, push to a new Space |

> **Note:** the source code, the 374 result parquet files (~900 MB via LFS), and the runtime config live on the **Hugging Face Space** — not on this GitHub repo (which only carries documentation).
>
> 🔗 **Code & data:** https://huggingface.co/spaces/jianlizhao/GeneDisease-MR-Browser

---

## A. Live app — zero install

1. Open **https://is.gd/virtual_gene_atlas** (or the direct https://jianlizhao-genedisease-mr-browser.hf.space/?embed=true)
2. Pick a category → pick a disease → choose a single gene / paste a list / upload a CSV
3. Click **Run / Show results**
4. Download the volcano PNG, KD/OE PNG, and result CSV

That's it. No login, no install, no GPU. The Space sleeps after 48 h of zero traffic; first visitor after sleep waits ~30 s for it to wake up.

---

## B. Run locally (clone from Hugging Face Space)

### B.1 Prerequisites

| Tool | Version | How to check |
|------|---------|--------------|
| Python | ≥ 3.10 | `python --version` |
| pip | ≥ 22 | `pip --version` |
| git | any modern | `git --version` |
| git-lfs | ≥ 2.10 | `git lfs version` |

**Disk:** ~1 GB for the parquet result files.

### B.2 Clone the HF Space repo

```bash
git lfs install
git clone https://huggingface.co/spaces/jianlizhao/GeneDisease-MR-Browser
cd GeneDisease-MR-Browser
git lfs pull                                # fetches the 374 parquet files (~900 MB)
```

### B.3 Install dependencies

```bash
# Optional but recommended: create a fresh virtual env
python -m venv .venv

# Activate it
# Windows PowerShell:
.\.venv\Scripts\Activate.ps1
# macOS / Linux:
source .venv/bin/activate

# Install
pip install -r requirements.txt
```

The dependencies (~200 MB):
- `gradio>=6.14`
- `pandas`, `pyarrow`, `numpy`, `scipy`
- `matplotlib`, `adjustText`

### B.4 Launch

```bash
python app.py
```

Watch for a line like:
```
Running on local URL:  http://0.0.0.0:7860
```

Open that URL in your browser. The app loads with all 374 diseases and 19,296 genes available immediately.

### B.5 Regenerate the data from scratch (optional)

If you want to recompute everything yourself (e.g., to swap in real `TwoSampleMR::mr()` calls instead of the deterministic mock):

```bash
# Step 1 — Pull all human HGNC genes (~17 MB)
python scripts/download_gene_list.py
# → data/raw/gene_list.csv (44,986 genes)
# → data/raw/gene_list_protein_coding.csv (19,296 genes)

# Step 2 — Pull GWAS Catalog studies + categories (~60 MB)
python scripts/download_gwas_catalog.py
# → data/raw/disease_list.csv (89,981 studies)

# Step 3 — Filter to a manageable subset
python scripts/filter_diseases.py --max-per-category 50
# → data/raw/disease_list_filtered.csv (~374 diseases)

# Step 4 — Batch compute (resume-safe; ~57 min for 7.2M pairs)
python scripts/batch_compute_all.py \
    --genes    data/raw/gene_list_protein_coding.csv \
    --diseases data/raw/disease_list_filtered.csv \
    --no-per-pair-figs \
    --top-n-aggregate 50

# Step 5 — Launch
python app.py
```

The batch is **resume-safe** — kill it (`Ctrl+C`) anytime and rerun; it skips diseases already done.

### B.6 Quick sanity check

```bash
python scripts/smoke_test.py
```
You should see `[SMOKE] all 9 pairs PASS`.

### B.7 Single-disease ad-hoc analysis (no app)

If you just want to generate one disease's volcano + KD/OE quickly:
```bash
python scripts/run_mr_pipeline.py \
    --genes data/raw/gene_list_alzheimer_demo.csv \
    --disease ebi-a-GCST005922 \
    --outdir outputs/my_run
```
Produces `outputs/my_run/{volcano,simulated_KO_OE}.png` + `result_table.csv`.

---

## C. Deploy your own HF Spaces copy

If you want your own browsable copy (your username, your tweaks, your branding):

### C.1 Hugging Face account
1. Sign up at https://huggingface.co (free)
2. Create a **Write** permission token at https://huggingface.co/settings/tokens
3. Install the CLI:
   ```bash
   pip install huggingface_hub
   hf auth login --token hf_YOUR_TOKEN --add-to-git-credential
   ```

### C.2 Duplicate the Space (recommended)
Fastest path: use Hugging Face's built-in **"Duplicate this Space"** button on
https://huggingface.co/spaces/jianlizhao/GeneDisease-MR-Browser
(top-right corner → "Duplicate Space"). It copies code + data + LFS files in one click.

Alternatively, manual path:

```bash
hf repo create <your-name>/<your-space-name> \
    --repo-type space --space_sdk gradio --exist-ok

# In your cloned copy of the original Space:
git remote add space https://huggingface.co/spaces/<your-name>/<your-space-name>
git push space main:main
```

### C.3 Edit YAML metadata
Tweak the frontmatter at the top of `README.md` (on the Space, not in this docs repo):
```yaml
---
title: <Your App Name>
emoji: 🧬
sdk: gradio
sdk_version: 6.14.0
app_file: app.py
short_description: <≤60 chars, no special chars>
---
```
Commit + push → HF auto-rebuilds.

For full deployment details — push failures, force-push safety, LFS budget — see [DEPLOY.md](DEPLOY.md).

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `ModuleNotFoundError: No module named 'gradio'` | You skipped `pip install -r requirements.txt` |
| `Port 7860 already in use` | Use `python app.py --server-port 7861` or kill the old process |
| Browser shows "Loading..." forever | Hard-refresh (Ctrl+F5). If still stuck, restart `python app.py` |
| `git lfs pull` says no files | Run `git lfs install` once first, then retry |
| Push to HF rejected: `file > 10 MiB` not in LFS | A log/output file slipped in. See DEPLOY.md "Common failures" |
| Smoke test fails | Check that `data/processed/*.parquet` files exist. If not, re-run section B.5 step 4 |
| App opens but the disease dropdown is empty | `data/processed/` has no parquet files. Either `git lfs pull` or regenerate from B.5 |
| Plot labels overlap / look ugly | `adjustText` not installed. `pip install adjustText>=1.1` |

---

## Project layout (for navigating the Hugging Face Space)

```
.
├── app.py                            # Gradio UI — the entry point
├── README.md                         # HF Spaces landing page
├── requirements.txt                  # Python dependencies
├── scripts/
│   ├── figures.py                    # Shared ggplot2-style plotting
│   ├── download_gene_list.py
│   ├── download_gwas_catalog.py
│   ├── filter_diseases.py
│   ├── batch_compute_all.py          # Main batch pipeline
│   ├── run_mr_pipeline.py            # Single-disease CLI
│   └── smoke_test.py                 # Pre-deploy sanity check
└── data/
    ├── raw/
    │   ├── gene_list.csv
    │   ├── gene_list_protein_coding.csv
    │   ├── disease_list.csv          # Optional, regenerable
    │   ├── disease_list_filtered.csv # Required by the app
    │   └── gene_list_example.csv     # Sample upload for users
    └── processed/
        └── disease_<gwas_id>_results.parquet  # 374 files (~900 MB via LFS)
```

---

## Performance reference

Numbers from one production run on a modest laptop (no GPU):

| Step | Time |
|------|------|
| Download HGNC genes | ~10 s |
| Download GWAS Catalog | ~30 s |
| Filter diseases | < 1 s |
| Batch compute (7.2 M pairs, mock) | **57 min** |
| Push to HF Spaces (first time, ~900 MB LFS) | ~3 min |
| First Space cold start | ~30 s |
| Per-search render (one disease, top 50 genes) | < 1 s |

Swap the mock `compute_one_pair()` in `scripts/batch_compute_all.py` for real `TwoSampleMR::mr()` calls when you have local GWAS sumstats — the rest of the pipeline doesn't change.

---

## Need help?

- **Public description** (what is this thing?) → [ATLAS.md](ATLAS.md)
- **Plot interpretation** (what does this figure mean?) → [INTERPRETATION.md](INTERPRETATION.md)
- **Workflow questions** (how to build a similar app) → [WORKFLOW.md](WORKFLOW.md)
- **Deployment troubleshooting** → [DEPLOY.md](DEPLOY.md)
- **Bugs / suggestions** → GitHub Issues on this repo

---

Last updated: 2026-05-11
