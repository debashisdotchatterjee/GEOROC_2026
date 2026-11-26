# GEOROC_2026

# GEOROC Bayesian compositional learning — empirical verification

Companion code to *How Much Can More Rocks Tell Us? Bayesian Compositional Learning for Tectonic Discrimination of Unseen Geological Provinces* (methodological draft). No real-data performance is claimed in this code package.

## Start in Google Colab

1. Open `GEOROC_Empirical_Verification.ipynb` in Google Colab: <https://colab.research.google.com/> → File → Upload notebook.
2. Choose a CPU runtime (the implementation is NumPy/SciPy/Scikit-learn; a GPU is not needed). The notebook writes the entire source locally; no second code upload is required.
3. In the Settings cell use `MODE = "pilot"` initially. Set `USE_DRIVE = True` if you want persistent archive downloads and checkpoints; Google will ask you to authorize mounting your own Drive. Otherwise the `/content` workspace is temporary.
4. Run all cells. At the end the notebook creates and downloads `georoc_results_pilot_outputs.zip` (or the corresponding research filename). If the browser blocks the download, use the final download cell or Colab Files pane. Return the **results ZIP**, not the code ZIP, for analysis.
5. Inspect `run_status.json`, `READ_ME_FIRST.txt`, `audit/`, `tables/feasibility_primary.csv`, `tables/fold_audit_primary.csv` and diagnostic tables before looking at model performance. A real-data failure does not substitute synthetic data. An actual run may produce only audit/simulation outputs if the minimum province count is not met.

## Modes and computational limits

| Mode | Purpose | Simulation | MCMC | Real data |
|---|---|---|---|---|
| `smoke` | Software verification only | 1 synthetic scenario, 1 rep, 2 test provinces/class | 2 chains, 10 warmup + 16 draws each | Optional; not for inference |
| `pilot` | First Colab investigation | 3 representative scenarios (Gaussian, t5/heterogeneous B, heterogeneous W), 1 rep each, 6 test provinces/class | 4 chains, 250 warmup + 350 draws each | Primary and major-only; 3 nested subsets per size |
| `research` | Prespecified comprehensive study | 36 core scenarios × 100 reps + 1 class-dependent-W sensitivity × 100 reps; 30 test provinces/class | 4 chains, 1000 warmup + 1500 draws each, automatic longer restart on training diagnostic failure | All prespecified sensitivities if enabled; 20 nested subsets per size |

Research mode is **not a realistic single Colab session**: the 3,700 simulation fits alone have many nested target predictions, and augmented sampling on difficult targets adds computation. Use persistent storage, set `MAX_SIM_JOBS` and `MAX_REAL_FOLDS` in the notebook to modest per-session numbers, and rerun with the same config and source. Those session budgets are not part of the statistical config. All completed scenario-replicates and folds are checkpointed. `MAX_REAL_FOLDS` is per variant; folds are interleaved across classes. Partial results are clearly reported, not treated as a complete study. No time-to-completion or convergence guarantee is made. A pilot uses the same likelihood and priors but much smaller Monte Carlo effort and is not publication-ready.

For a local CPU run:

```bash
python -m pip install -r requirements.txt
python georoc_analysis.py --mode smoke --no-real --root smoke_results
python georoc_analysis.py --mode pilot --no-sensitivity --root pilot_results --cache archive_v13
```

Do not reuse an output root with a changed scientific config or edited code. Choose a new root. To run just simulations or just real data, use `--no-real` or `--no-sim` and a separate root; the notebook also offers these switches. For longer experiments the notebook accepts per-session budgets. See source for all configuration fields.

## Data sources and curation

The notebook requests **version 13.0** of each GRO.data Dataverse compilation through the documented Native API, then downloads the original-format CSV files by file ID. It saves file identifiers, SHA-256 hashes and repository checksums (verified for non-ingested files), source metadata and citation footers. It never silently switches to a latest version. Source DOIs:

- OIB, Ocean Island Groups: <https://doi.org/10.25625/WFJZKY>
- CFB, Continental Flood Basalts: <https://doi.org/10.25625/WSTPOX>
- VAB, Convergent Margins: <https://doi.org/10.25625/PVFZCE>
- Dataverse version API: <https://guides.dataverse.org/en/latest/api/native-api.html>
- GEOROC archive information: <https://georoc.eu/georoc/precompiled/>
- Original sources must also be credited: <https://georoc.mpch-mainz.gwdg.de/georoc/cite.asp>.

The remote archive was not reachable in the development container (network/DNS unavailable). Its documented schema and API were checked, but a live v13 file download/parser run remains untested. Colab will attempt the download. If the archive requests acceptance of data-use terms, visit the DOI pages, accept the terms if appropriate and manually download the same version 13 files. Set `LOCAL_ARCHIVE` to the directory containing the CSVs. Version metadata must still be retrievable or placed as `{DOI_suffix}_v13.0.json` (the Dataverse API response's `data` object) inside `CACHE`; manual files are matched to the pinned manifest, not an arbitrary directory. The script does not bypass authentication or use fabricated data. Unrecognized schema/malformed CSV records trigger an explicit error; send the resulting ZIP and error log rather than silently discarding records. Raw downloads remain in the cache (not duplicated inside results ZIP); the analytic candidates, audit and source manifest are exported.

### Geological review is a required step, not an algorithmic claim

The first run uses **provisional archive group labels**: split filenames merged by name, plus conservative metadata agreement. It does not claim that these groups represent independent geological provinces or that archive labels are independently corroborated. The code exports `audit/province_review_template.csv`, `audit/candidate_row_metadata.csv.gz`, `audit/row_review_template.csv` and original references. For reviewed analysis:

- Review every included file against geological metadata and original publications. Assign coherent `province` names (merge related subunits), mark `include=0` for ambiguous/ineligible files; for included files set `verified=1` and provide an `evidence` citation or rationale. Keep all rows of the template (including excluded files).
- For exceptional individual records, use the row-review template: exact `row_id`, `include` 0/1, nonempty `evidence`, optional canonical `specimen_key` and pipe-delimited `reference_keys`. This can reconcile specimen aliases/citation variants and confirm metadata disagreements; it cannot bypass chemical quality filters or invent measurements. Review cross-file duplicate and citation matching, which is heuristic, and correct alias keys before inference. Row metadata IDs are data-record indices, not physical line numbers.
- Set `CURATION_FILE`, optional `ROW_REVIEW_FILE`, and `REVIEWED = True` **only after** that review. Use a new results root. The review flag records your review, not independent verification by the code. There is no automatic guarantee against all geographical or bibliographic overlap.

The manuscript's primary gate is ≥10 eligible independent provinces per class, ≥5 unique rocks per province. The program refuses the primary fit if this fails; the major-only analysis can still run if it meets its own gate. No artificial splitting of provinces is allowed. Each fold removes training records with reference-key overlaps with the held-out province, then excludes training groups with <5 rocks; if too few training provinces remain, the fold is marked failed. Complete predictions are required before final claims. The publication key heuristic uses explicitly identified global IDs, DOI or normalized author/year/reference text. It can overblock author-year collisions or miss citation variants; manual review is essential.

### Preprocessing decisions

8 major oxides in wt%, 6 trace elements in ppm (converted to wt% before ILR), positive uncensored complete cases, documented whole-rock volcanic material, documented altered/ambiguous alteration excluded, unknown alteration retained and audited. Fe2O3T preferred; FeOT × 159.688/(2×71.844); both separately reported FeO and Fe2O3 combined only when total unavailable; discordant totals (>5% relative difference) rejected. Major-oxide total 85–105 wt% **before closure** and normalized SiO2 45–52 wt%. These oxide-total bounds, iron tolerance, publication-year cutoff (2000), conservative setting codes, and fixed ML hyperparameters are explicit implementation decisions not fixed in the manuscript. No arbitrary zero replacement. Metadata and publication records are not features. Curation and retention are audited. The `publication_year` is a heuristic first year from citation text; this sensitivity needs source-level confirmation.

### Mathematics and evaluation

- Helmert ILR (`log(a) @ H`) and **province-balanced** training-only centering and scalar scaling, as specified in main.tex.
- Gibbs sampler for the exact normal hierarchical model with class-specific B_k and common W. Priors: mu_k ~ N(0,25I); B_k,W ~ IW(d+3,I), with proper sensitivity alternatives. Use sufficient statistics; no neural network is imposed.
- Joint target likelihood C_n(W,T) N(mean; mu_k,B_k+W/n); integrate each class likelihood across training posterior draws before normalizing (log-sum-exp). Do not average within-draw normalized probabilities. Importance ESS for every class and mixture, batch-means relative Monte Carlo SE, and augmented Gibbs fallback under the joint posterior with unlabelled target. Fallback class probabilities are Rao–Blackwellized under that updated posterior. Both kinds of failure remain in exported diagnostics.
- Independent-rock Bayesian ablation uses a conjugate normal-inverse-Wishart prior (kappa0=0.02, nu0=d+3, S0=I, m0=0); the class-specific joint likelihood integral is calculated **exactly** via NIW normalizers, not plug-in multiplication. The prior marginal covariance of the mean is 25I at the primary specification.
- Class-balanced logistic regression and RF (fixed C=1, min leaf 5, sqrt features; preset trees); arithmetic average of rock probabilities, not original SGTPPR. RF exact zeros floored at 1e-15 solely for score stability. Equal target priors. No tuning or probability calibration performed; any future tuning must be nested by province.
- Scores: class-balanced NLL (primary), ordinary NLL, Brier, balanced accuracy, reliability, confidence, paired class-stratified descriptive bootstrap. Test subsets are nested; scores are averaged within province before class aggregation. Random rock split is an easier known-province task, reported separately. Oracle exact Gaussian rules are shown only for Gaussian simulations; no incorrect Gaussian oracle is claimed for t5 stress.
- Rank-normalized split Rhat, bulk ESS (Geyer initial positive/monotone sequence), representative trace summaries, per-target weight ESS and batch-means relative MCSE. Gates Rhat ≤1.01, ESS ≥400, relative MCSE ≤0.20, importance ESS ≥max(50, 2% of draws) in normal presets; pilot/smoke often fail. Research training sampler retries once with double draws/warmup and augmented prediction retries up to configured maximum. These are diagnostic heuristics, not convergence proofs; if still flagged, do not publish those fits as validated. More MCMC may be required, and the Gaussian model itself may be inadequate.

## Output ZIP structure

- `config.json`, `environment.json`, `run_status.json`, `run.log`, `numerical_self_tests.json`, `READ_ME_FIRST.txt`, executed source copy
- `audit/`: archive metadata/manifest, source references, schemas, curation templates, analytic candidates, errors if any
- `tables/`: CSV and readable HTML (small tables), data/feasibility audits, fold scores, raw predicted probabilities/log probabilities, simulation scores and MCSE, MCMC/importance diagnostics, paired intervals, reliability, sensitivities
- `figures/`: print-ready vector PDFs plus 300-dpi PNGs, e.g. data audit, province-level ILR PCA, performance, evidence curves, confusion matrices, reliability, confidence, covariance and trace checks, theoretical saturation and simulation curves
- `checkpoints/`: scenario-replicate and fold outputs, diagnostic files, fingerprints; retain for resumability


