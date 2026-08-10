# CLAUDE.md — Primer Tm & Annealing Calculator (IRP-211)

Onboarding for a Claude agent taking over this project. Read this fully before editing.

## What this project is

A single-file, browser-based **primer melting-temperature (Tm) and annealing-temperature calculator**, built for the IRP-211 mammalian pMHC-I / TCR display project (designing and validating TCR/HLA sequences and oligo pools for an NGS workflow). It computes nearest-neighbor Tm the way SnapGene/IDT/NEB do, but is tuned for **KAPA HiFi HotStart** (Roche) and also predicts annealing temperatures for **NEB Q5 / Phusion / Phire**, Platinum SuperFi, and standard Taq. It designs primers in batch from FASTA/CSV and maps adapter-tailed primers onto templates.

The scientific goal: design forward primers across TCR V-gene sets (e.g. TRAV/TRBV) that all hit a common target Tm with minimal spread, for a synthetic oligo pool.

## The one file that matters

**`260715_primer_tm_calculator.html`** is the entire application and the single source of truth — HTML, CSS, and JavaScript in one self-contained file. Open it in any browser; no server, no build step, no internet needed. Everything (Tm engine, three tabs, CSV export) lives inside its one `<script>` block.

**`eng*.js` files are NOT source.** They are throwaway extracts of the HTML's script block created only for Node-based verification (see below). Never edit them and never treat them as authoritative — regenerate them from the HTML each time. If they drift from the HTML, delete and re-extract.

`designed_Vgene_primers_60C.csv` is an example output (387 TRAV/TRBV forward primers designed to 60 °C).

## How the app is structured

Top of page: **shared conditions panel** — reaction presets (KAPA HiFi / SnapGene / IDT), monovalent/Mg²⁺/dNTP/oligo concentrations, salt-correction model, a **polymerase selector** (drives annealing Ta), and a **"Calibrate SnapGene Tm" checkbox**. These settings feed every tab.

Three tabs:
1. **Oligo Tm** — single oligo + batch paste. Shows Tm, GC%, ΔG, suggested Ta for the selected polymerase, and an 8-point annealing gradient.
2. **Primer → template** — maps one or more primers (with optional 5′ adapter tails) onto a template strand, finds the longest perfect 3′-anchored annealing region, reports annealing-region vs full-primer Tm, and draws the duplex.
3. **Primer designer** — anchors a primer end (5′ or 3′) at a base or range, grows it until Tm hits a target, in batch from paste or uploaded FASTA/CSV. Summary stats + CSV.

The JS is organized by comment banners you can grep for:
`// ===== VALIDATED NN ENGINE`, `// ===== UI helpers`, `// ===== OLIGO TAB`, `// ===== TEMPLATE TAB`, `// ===== PRIMER DESIGNER`, `// ===== export`, `// ===== wire up live updates`.

## The Tm engine (do not break this)

Located under `// ===== VALIDATED NN ENGINE`. Key pieces:

- `NN` — SantaLucia 2004 unified nearest-neighbor ΔH/ΔS parameters. `INIT_GC`/`INIT_AT` are the **Allawi & SantaLucia 1997** terminal-initiation terms (= Biopython `DNA_NN3`). We deliberately do NOT use the SantaLucia & Hicks 2004 (`DNA_NN4`) init scheme; the small residual difference vs SnapGene is handled by empirical calibration instead (see below).
- `thermo(seq)` → `{dH,dS}`. Degenerate (IUPAC) bases are handled by **averaging** ΔH/ΔS over all base combinations at each nearest-neighbor step and at the terminal init — this matches SnapGene's stated convention.
- `tmFromThermo(dH,dS,Cstrand,selfc)` — concentration term `k = selfc ? Cstrand : Cstrand/2`. **The `/2` is important and correct**: it makes a per-strand primer concentration match the SnapGene/IDT/Biopython convention (`C_T/4` for equal strands). An earlier bug used `/4` and was ~1 °C off — do not "simplify" this back.
- `saltCorrect(...)` — Owczarzy 2004 (monovalent) and Owczarzy 2008 (mono+divalent) corrections. **Concentrations here must be in mol/L** (the code divides mM by 1000). An earlier bug passed mM and was ~1–1.5 °C off on the KAPA path — watch for this.
- `dSsalt(...)` — SantaLucia 1998 monovalent entropy correction (used by the `santalucia` model).
- `analyze(seq, opts)` → `{seq,N,gc,dH,dS,dG,tm,selfc}`. `opts = {mono, mg, dntp, oligo, model}`, `model ∈ {"owczarzy","santalucia"}`. This is the single entry point for a Tm.

IUPAC support: `clean()` **preserves** degenerate codes (`WSMKRYBDHVN`) and maps U→T — never strip them. `baseSet`, `comp`/`revcomp`, and `isSelfComp` are all degenerate-aware. `gcFrac` returns fractional GC. Self-complementarity is only tested for fully-concrete sequences.

## Conventions & presets

- **KAPA_OPTS** = 60 mM mono, 2.5 mM Mg²⁺, 1.2 mM dNTP, Owczarzy 2008. **SNAP_OPTS** = 50 mM Na⁺, no Mg, SantaLucia 1998. Default oligo 250 nM per strand.
- **Polymerase annealing (`polAnneal`)**: Q5/Phusion/Phire = SnapGene Tm **+6–12 °C** (start +9); Platinum SuperFi +4–8; Taq −5–0; KAPA = near the KAPA-condition Tm (gradient Tm−5..+3, 65 °C fallback). Q5/SuperFi capped at **72 °C** (NEB). These are per SnapGene/NEB/Thermo guidance — annealing for dsDNA-binding-domain enzymes is *above* the primer Tm.
- **SnapGene calibration** (`SNAP_CAL`, `snapTm`): when the checkbox is on, `Tm_SnapGene ≈ raw − 0.995 + 3.194·(GC fraction)`, displayed rounded to nearest integer. This was **fit to 9 user-measured SnapGene values** (TRAV/TRBV primers, 20–40 nt, ~60 °C; reproduces all 9 exactly, RMS 0.26 °C). The GC term reflects where SnapGene's salt handling diverges from ours. It is empirical and validated only near that range — treat as approximate elsewhere. `snapTm()` (not raw `analyze(..,SNAP_OPTS)`) is what all SnapGene displays, the designer's SnapGene target, and the Q5/SuperFi/Taq annealing math should call, so calibration flows everywhere consistently. If more measurements arrive, refit `SNAP_CAL.a` and `SNAP_CAL.c`.
- Template/primer coordinates are always reported on the **top strand as entered**, regardless of which strand a primer anneals to.
- Mapping anchors on the primer 3′ end and returns the **longest** perfect annealing region; a coincidental extra base into the adapter that still pairs is correct, not a bug.

## Verification is mandatory

Every change to the Tm math must be re-checked against **Biopython** (independent implementation of the same models) before you ship. Workflow used throughout this project:

1. Extract the script (or a function group) from the HTML with a Python regex, write to a temp `eng.js`, append a `module.exports`.
2. Run in Node and compare to `Bio.SeqUtils.MeltingTemp.Tm_NN`.

```bash
pip install biopython --break-system-packages -q
# Biopython equivalences to our modes (use dnac1=dnac2=250):
#   santalucia model  -> nn_table=DNA_NN3, saltcorr=5   (matches to <=0.06 C)
#   owczarzy monovalent-> nn_table=DNA_NN3, saltcorr=6
#   owczarzy + Mg (KAPA)-> nn_table=DNA_NN3, saltcorr=7  (matches to <=0.05 C)
```

Also always run `node --check` on the extracted full `<script>` to catch syntax errors, and confirm the 9-point SnapGene calibration still returns 9/9 after rounding. Known-good baselines: raw SnapGene mode ≤0.06 °C vs Biopython; KAPA/Owczarzy path ≤0.05 °C; degenerate primer `...CAGTTRAGCTGGG...` R-Tm sits between its A and G variants.

Regex extraction patterns that work against the current file:
- Engine: `<script>(.*?)// ================= UI helpers`
- `(function parseSeqs.*?)// ================= OLIGO TAB`
- `(function parseTemplates.*?)// ================= export`
- Mapping: `(// two bases.*?)function renderAlignment`

## House rules / gotchas

- Keep it a **single self-contained HTML file**. No external libraries, no build tooling, no `localStorage`/`sessionStorage` (unsupported in the target environment). Only CDN allowed if ever needed: cdnjs.
- Live updates: most inputs re-run via `input`/`change` listeners calling `runAll()`. The designer intentionally runs on its button + parameter changes + file upload, **not** on every keystroke of the big template box (avoids lag on large FASTA).
- When you add a column, update the table header, the row builder, AND the CSV header/row arrays together — they must stay column-aligned. Verify counts.
- Two-state model only: no hairpin/self-dimer/secondary-structure prediction. If asked for that, it's a new feature, not a tweak to `analyze`.
- Biopython's `Tm_NN` has `selfcomp=False` by default and does not auto-detect palindromes — don't be misled when comparing self-complementary cases.

## Project context

Parent project: IRP-211_MAMMALIAN-pMHC-I_TCR_LOL — a mammalian display system expressing TCRs and HLA+peptide, sorted by flow cytometry and read out by NGS. TCR chains come from a synthetic oligo-pool library (designed with this tool) or from sequenced human PBMC repertoires. This calculator supports the oligo/primer design side: consistent-Tm primers for KAPA HiFi and Q5 amplification of V-gene panels.

## Likely next requests (already scoped in-chat)

- An in-tool panel to paste new (primer, SnapGene Tm) pairs and **re-fit the calibration** live.
- Per-variant Tm range for degenerate primers (min/max across combinations).
- A length cap with "best achievable Tm" fallback in the designer, and/or a per-family consensus primer mode.
- Self-dimer / hairpin ΔG screening.
