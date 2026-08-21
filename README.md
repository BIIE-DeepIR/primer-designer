# primer-designer

Primer Tm & annealing-temperature calculator for PCR primer design — nearest-neighbor thermodynamics (SantaLucia 2004) with Owczarzy salt correction, tuned for KAPA HiFi HotStart and cross-calibrated against SnapGene/IDT.

Single self-contained HTML file, no build step, no server, no internet required — open [`260715_primer_tm_calculator.html`](260715_primer_tm_calculator.html) in any browser.


## 🌐 Live Public Website
https://biie-deepir.github.io/primer-designer/260715_primer_tm_calculator.html

## Features

- **Oligo Tm** — single or batch sequence Tm/GC%/ΔG, an 8-point annealing gradient, and a **Dual Primer (Fwd + Rev)** mode for batch Tm/Ta on primer pairs (CSV import/export, downloadable template, pasteable/editable table with spreadsheet-style multi-cell paste).
- **Primer → template** — maps primers (with optional 5′ adapter tails) onto a template strand, finds the longest perfect 3′-anchored annealing region, and reports annealing-region vs. full-primer Tm with a duplex diagram.
- **Primer designer** — grows a primer from an anchored 5′/3′ position until it hits a target Tm, in batch from pasted sequences or an uploaded FASTA/CSV, with summary stats and CSV export.

Annealing-temperature (Ta) guidance is provided per polymerase: KAPA HiFi HotStart, NEB Q5, Phusion/Phire, Platinum SuperFi, and Taq — each following vendor-specific offset conventions (see the in-app "Method, assumptions & verification" panel for details and citations).

## Usage

Just open the HTML file. Reaction condition presets (KAPA HiFi / NEB Q5 / SnapGene / IDT) set monovalent, Mg²⁺, dNTP, oligo concentration, and salt-correction model in one click; all three tabs share these conditions.

## Development

All logic lives in the one HTML file's `<script>` block, organized under comment banners (`VALIDATED NN ENGINE`, `OLIGO TAB`, `TEMPLATE TAB`, `PRIMER DESIGNER`, etc.). Changes to the Tm math are verified against Biopython's `Bio.SeqUtils.MeltingTemp.Tm_NN` — see [`CLAUDE.md`](CLAUDE.md) for the full verification workflow and engine internals.
