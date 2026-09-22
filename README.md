# PepBench

[![License](https://img.shields.io/github/license/ProteinQure/pepbench?label=license)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/ProteinQure/pepbench?display_name=tag&sort=semver)](https://github.com/ProteinQure/pepbench/releases)

PepBench is a set of *de novo* designed peptide binder backbones against 12 protein
targets, built to decouple sequence recovery from design-success metrics. Because the
binders are generated rather than observed, there is no native sequence to recover:
sequence-recovery numbers are undefined here, and evaluation relies on structural
measures instead.

This repository holds the benchmark's **peptide backbone templates** — the inputs to an inverse-folding method.

## Contents

```
templates/<TARGET>/len<L>/pick<N>_<TARGET>_len<L>.pdb
```

Each file is a two-chain complex:

- **Chain A** — the designed peptide backbone, `len08`, `len16` or `len24` residues.
  All residues are `GLY` and carry backbone atoms only (`N`, `CA`, `C`, `O`), so the
  file specifies a backbone and a pose, never a sequence.
- **Chain B** — the receptor, renumbered `1..N`. Its sequence is identical across every
  pick and length of a target.

Files are plain `ATOM` records with a placeholder `CRYST1` line, no hydrogens and no
`SEQRES` header. Each complex sits in its own coordinate frame; the receptors of one
target superpose to within a few tenths of an Å but are not pre-aligned to a common
frame.

## Quick use

1. Select a template from `templates/<TARGET>/len<L>/`.
2. Give the two-chain PDB to an inverse-folding method: design residues for chain A
   while using chain B as the fixed receptor context.
3. Refold the designed peptide–receptor complex with your structure predictor of
   choice (e.g., Boltz-2 or OpenDDE).
4. Assess target-aligned peptide self-consistency RMSD: align the predicted and
   intended complexes on chain B, then calculate chain A backbone RMSD without a
   second alignment on chain A.

Aligning on the target alone means scRMSD reports consistency of both the peptide
backbone and its pose relative to the target, rather than peptide shape in isolation.
This is the standard design–refold self-consistency principle, adapted to
peptide–protein complexes.

## Targets

Twelve targets spanning a diverse set of protein families, each taken from a solved
peptide–protein complex and stripped to the receptor chain. The receptor is chain B
because BoltzGen's de novo path assigns the designed binder to chain A.

| PDB | UniProt | Protein |
| --- | --- | --- |
| 1FGL | P62937 | Peptidyl-prolyl cis-trans isomerase A (PPIA / cyclophilin A) |
| 3JQ5 | P60045 | Acidic phospholipase A2 3 |
| 3N00 | P20393 | Nuclear receptor subfamily 1 group D member 1 (NR1D1 / Rev-erbα) |
| 3O2M | P45983 | MAPK8 (JNK1) |
| 4RRV | O15530 | PDPK1 (PDK1) |
| 4Y7R | P61964 | WDR5 |
| 6CDG | Q8IVV7 | GID4 |
| 6O33 | Q13526 | PIN1 |
| 6YOO | Q9H0R8 | GABARAPL1 |
| 7OUN | Q9NZQ7 | CD274 (PD-L1) |
| 7YKH | Q9WVQ1 | MAGI2 (PDZ0–GK fragment) |
| 9CDT | Q07820 | MCL1 |

## How the backbones were made

**Generation.** Backbones were generated with BoltzGen (Stark et al., 2025; `peptide-anything` protocol) in de novo mode, conditioned on the receptor structure and a hotspot subset. For each length:target pair, 10 independent configurations were launched, each drawing a fresh subset of 3–6 hotspots constrained so all chosen residues are mutually reachable by a binder of the requested length (every pairwise Cα distance within a length-scaled cutoff). Binder
length was fixed per run. This produced on the order of 10³ candidate backbones per length:target pair.

**Hotspots.** Binding-site residues were assigned per target by an automated literature-and-precedent pipeline that resolves the UniProt accession, retrieves related PDB structures and their literature, and scores every receptor residue by how much precedent supports it as a contact. Filtered sets were compared against the contacts made by the peptide in the original complex as a sanity check on site placement.

**Selection.** Candidates were clustered and filtered down to the 10 per length:target pair released here, giving 353 structures — 10 per pair except those that did not reach 10 distinct clusters (len08:1FGL with 7 and len16:7OUN with 6).

## Evaluation

The steps in [Quick use](#quick-use) leave several choices open, and different choices
give different numbers. This section records the evaluation behind the PepBench results
in the AtomWeaver preprint, in enough detail to reproduce them.

**Refolding.** The whole complex was refolded: the structure predictor received the
designed peptide sequence, the target protein (receptor) sequence, and the target MSA. The structures of both chains were then predicted. We used [OpenDDE](https://github.com/aurekaresearch/OpenDDE) as the structure predictor.

**Superposition.** Each predicted complex was superposed onto its template on the
backbone atoms (`N`, `CA`, `C`, `O`) of the receptor chain, over all receptor residues —
not Cα alone, not all heavy atoms, and not an interface subset. The template was held
fixed and the prediction was the mobile structure.

**scRMSD.** The receptor-derived rotation and translation were applied to the predicted peptide, and scRMSD is the peptide backbone self-consistency RMSD (`N`, `CA`, `C`, `O`) against the template peptide, with no second alignment on the peptide. Sidechains never enter the number, so *canonical and
non-canonical designs are scored on the same footing*.

**Residue correspondence.** Residues were matched by position along the chain rather
than by residue number, since predictors routinely renumber chains from 1. A predicted peptide therefore had to carry the same residue count as its template.

**Unscorable designs.** A design was unscorable if its predicted peptide had a different
residue count from the template, or if any peptide residue was missing one of the four
backbone atoms — the common case for non-canonical residues written with ligand-style
atom naming, which have no identifiable backbone. Residue names were never compared
between prediction and template: the sequence was redesigned, so disagreement is
expected. Unscorable designs were counted as failures at every threshold and reported
alongside the results, since dropping them silently inflates success counts.

**Receptor diagnostic.** The receptor backbone RMSD after superposition was reported as
well. It separates a genuinely misplaced peptide from a prediction whose receptor was
itself mispredicted, which is otherwise invisible in the peptide number.

**Thresholds.** Two were used: 2 Å as the strict criterion and 5 Å as the lenient one.
The looser threshold reflects that peptides are floppier and less rigid than folded
domains, so a design may recover the correct binding epitope while retaining
conformational inconsistency that a strict cutoff rejects. Neither threshold is
calibrated against binding data.

**Reporting.** Results were reported as raw counts per (target, length) cell — designs
below each threshold, with unscorable designs counted as failures — rather than as a
single pooled figure, together with the number of designs generated per template and any
deduplication of identical sequences, which changes what a count is counting.

## Scope and limitations

**PepBench measures self-consistency, not affinity.** A design that refolds to the
intended backbone and pose is self-consistent with the template it was designed for.
The set carries no binding data, and no experimental validation, so scRMSD pass rates should not be reported as hit rates or compared against measured affinities.

**There are no peptide sequences, by construction.** The backbones are de novo
generated. Chain A is a poly-glycine placeholder carrying only backbone atoms. Sequence-recovery metrics are therefore undefined on PepBench.

**The receptors are real proteins and may overlap your training data.** Each receptor
is taken from a solved complex, so a model may have seen it or a close homologue
during training. The peptide backbones have no such counterpart, but a strong result
on a given target should be read with that target's training overlap in mind. Overlap
is specific to each model's corpus and is worth measuring against your own.

**Report results stratified by target and length.** Pooled results hide axes that was
built to vary along: target difficulty and peptide length. We strongly recommend against 
pooling the results into a single statistic, as it is expected to be noisy.

## Citation

PepBench was introduced in the AtomWeaver preprint:

```bibtex
@article{kitaygorodsky2026atomweaver,
  author    = {Kitaygorodsky, Alexander and Hostallero, David Earl and
               Broom, Aron and Layne, Elliot and Hwang, Sungwon and
               Kanawaty, Ashlin K. and Babej, Tomáš and
               Butterfoss, Glenn L. and Fingerhuth, Mark},
  title     = {{AtomWeaver}: Multi-Component Flow Matching with a Structured
               Geometric Prior Facilitates Non-Canonical Peptide Design},
  journal   = {bioRxiv},
  year      = {2026},
  publisher = {openRxiv}
}
```
