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

Chain A is a backbone-only poly-glycine placeholder, so PepBench is not appropriate
for sequence-recovery evaluation.

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

## Leakage

The receptors are real proteins, so a model evaluated on PepBench may have seen close homologues in training. The peptide backbones are generated and have no native counterpart, but a strong result on a given target should still be read with the receptor's training overlap in mind.

## Intended use

Design sequences for chain A conditioned on chain B, then assess them by refolding the designed complex and measuring target-aligned peptide self-consistency RMSD: superpose predicted and intended complexes on the target alone, then measure peptide backbone
RMSD without a second alignment on the peptide. That reports consistency of both the peptide backbone and its pose relative to the target, rather than peptide shape in isolation — the standard design–refold self-consistency principle, adapted to peptide–protein complexes.

## Citation

PepBench was introduced in the AtomWeaver preprint:

```bibtex
@article{kitaygorodsky2026atomweaver,
  author    = {Kitaygorodsky, Alexander and Hostallero, David Earl and
               Butterfoss, Glenn L. and Broom, Aron and Fingerhuth, Mark},
  title     = {{AtomWeaver}: Multi-Component Flow Matching with a Structured
               Geometric Prior Facilitates Non-Canonical Peptide Design},
  journal   = {bioRxiv},
  year      = {2026},
  publisher = {Cold Spring Harbor Laboratory}
}
```
