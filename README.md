# DeepEP-on-EFA Guidance Site

Self-contained static site (no build step) — measured guidance on whether to use DeepEP
MoE expert-parallel all-to-all, or a dense/allgather alternative, on AWS EFA.

## Deploy to GitHub Pages
Copy this directory to a public repo (or `docs/` on `main`), enable Pages → serve from that
folder. `index.html` is the landing page. All assets are relative; no external CDN/JS deps.

## Structure
- `index.html` — overview + headline measured numbers + one-paragraph answer
- `sections/01-what-is-moe.html` — MoE primer (router, dispatch/combine, EP)
- `sections/02-efa-vs-infiniband.html` — the fabric math (SRD vs RC, the proxy tax)
- `sections/03-scaling.html` — single/multi-node, conc/world-size/expert sweeps (charts)
- `sections/04-microbenchmarks.html` — transport GB/s + real AIPerf inference numbers
- `sections/05-alternatives.html` — dense, UCCL, pplx, IBGDA: do they serve, how fast
- `sections/06-solution-arch.html` — what works today + internal/customer decision trees
- `calculator/index.html` — interactive DeepEP-vs-alternative verdict calculator
- `calculator/MEASURED-DATA.md` — the sourced measured-data table behind everything
- `assets/site.css` — shared dark theme

## Provenance
Every claim is tagged MEASURED / DERIVED / ASSUMED. Calibrated to the efa-gda campaign:
2× p5en.48xlarge, AWS EFA, Qwen3-30B-A3B-FP8 EP16, AIPerf 0.10.0. Sanitized — no internal
IPs, registry IDs, or hostnames.
