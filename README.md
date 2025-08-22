
# TEMPO –  *T*CR–*E*pitope *M*otif‑based *P*redictor *O*f interactions


## Overview

**TEMPO** – a motif-based predictor of TCR–epitope interactions – estimates whether a T-cell receptor (TCR) can recognize a given epitope (a peptide presented by a specific MHC allele). The prediction is based on scoring V-gene, J-gene, and CDR3 features against a pre-computed motif model. For each input TCR, the tool outputs:

- **`perc_rank`** – percentile rank (lower = stronger binder)
- **`problem`**    – quality‑control flags (gene name typos, non‑AA symbols, etc.)

By default **both α and β chains** are evaluated (`--chain AB`). Use `--chain A` or `--chain B` to score a single chain.

Simply download the TEMPO executable for your machine architecture, and you are ready to use it.

## Quick Start 

```bash
# See which epitope motif models are available 
./TEMPO list_epitopes

# Predict which TCRs in the test/test.csv file are specific to the A0201_LLWNGPMAV epitope
./TEMPO predict ./test/test.csv ./test/output_predictions.csv A0201_LLWNGPMAV

# Predict using the α-chain only
./TEMPO predict ./test/test.csv ./test/output_predictions_chainA.csv A0201_LLWNGPMAV --chain A

# Large dataset example (100k rows)
./TEMPO predict ./test/test_100k_tcrs.csv ./test/out_100k.csv A0201_GILGFVFTL
```

> **Notes**
-   **Argument order matters** for `predict`: 
    `./TEMPO predict <input_tcr_file.csv> <output_predictions.csv> <epitope_model> [options]
`     
-   **Epitope motif models** use the format: `HLAallele_epitopeSequence`  
    Example: `A0201_LLWNGPMAV` → the epitope **LLWNGPMAV** presented by **HLA-A*02:01**.
    
-   **TCR input file headers** must match exactly (case-sensitive):
    -   Dual-chain (`AB`): `cdr3_TRA,TRAV,TRAJ,cdr3_TRB,TRBV,TRBJ`
    -   Single-chain (`A`): `cdr3_TRA,TRAV,TRAJ`        
    -   Single-chain (`B`): `cdr3_TRB,TRBV,TRBJ`
    
- **Species selection**: two species are supported — `HomoSapiens` (default) and `MusMusculus`.  
  Use `--species <NAME>` to select the species (affects gene lists, QC, and model availability).  
  Example (mouse): `./TEMPO predict input.csv out.csv H2Kb_SIINFEKL --species MusMusculus`

## Command‑Line Reference

TEMPO provides three sub‑commands:

- `list_epitopes` – show the list of embedded epitope motifs
- `correct_VJ`    – convert V/J gene labels to canonical IMGT
- `predict`       – score TCRs against a specific epitope motif **(main workflow)**

```
TEMPO list_epitopes [--csv <FILE>] [--tsv <FILE>]
TEMPO correct_VJ   <input.csv> <output.csv> [--species NAME] [--chain AB|A|B]
TEMPO predict      <input.csv> <output.csv> [epitope_ID]
                    [--motif FILE] [--species NAME] [--chain AB|A|B] [--no-qc]
```

### A) List available epitope motifs

```text
TEMPO list_epitopes [--csv <file>] [--tsv <file>]
```

Prints a table of all **embedded** motifs and exits. You can also export the list.

| Option       | Description                                    |
|--------------|------------------------------------------------|
| `--csv FILE` | Write the list as CSV to `FILE`.               |
| `--tsv FILE` | Write the list as TSV to `FILE`.               |

---


### B) Correct V/J gene names in raw input (`correct_VJ`)

`./TEMPO correct_VJ <input_raw.csv> <output_clean.csv> [--species NAME] [--chain AB|A|B]` 

Prepares a raw TCR file for prediction by converting V/J gene labels to canonical **IMGT gene names** (allele suffixes removed, common typos corrected)

| Option      | Default       | Description                                                 |
|-------------|---------------|-------------------------------------------------------------|
| `--species` | `HomoSapiens` | Choose species-specific gene lists (e.g. `HomoSapiens`, `MusMusculus`). |
| `--chain`   | `AB`          | Chains to process: αβ (`AB`), α only (`A`), or β only (`B`). |

---

### C) Predict TCR‑epitope interactions

```text
TEMPO predict <input.csv> <output.csv> [epitope_ID]
              [--motif FILE] [--species NAME] [--chain AB|A|B] [--no-qc]
```

Provide **either** a positional `epitope_ID` (uses embedded model) **or** `--motif FILE` (explicit `.bin` motif).

| Required                         | Description                                                    |
|----------------------------------|----------------------------------------------------------------|
| `<input.csv>` `<output.csv>`     | Input and result paths.                                        |
| `epitope_ID` (positional)        | E.g. `A0201_LLWNGPMAV`.                                       |
| **OR** `--motif FILE`            | Use a specific encrypted motif file (`.bin`).                  |

| Optional      | Default       | Purpose                                                    |
|---------------|---------------|------------------------------------------------------------|
| `--species`   | `HomoSapiens` | `HomoSapiens` or `MusMusculus`.                            |
| `--chain`     | `AB`          | Score αβ (`AB`), α‑only (`A`) or β‑only (`B`).             |
| `--no-qc`     | off           | Score all rows even if they fail QC checks.                |


## Format of the Input / Output files

### Input CSV (AB)

| Column     | Example             | Required |
|------------|---------------------|----------|
| `cdr3_TRA` | `CAVRDSNYQLIW`      | ✓        |
| `TRAV`     | `TRAV12-2`          | ✓        |
| `TRAJ`     | `TRAJ33`            | ✓        |
| `cdr3_TRB` | `CASSLGQDTQYF`      | ✓        |
| `TRBV`     | `TRBV19`            | ✓        |
| `TRBJ`     | `TRBJ2-7`           | ✓        |

(Any extra metadata columns are preserved and passed through.)

### Output CSV

| Column        | Description                                         |
|---------------|-----------------------------------------------------|
| `problem`     | Quality control flags (if any).                                  |
| `perc_rank`   | Lower = stronger predicted binder.                  |
| *(original)*  | All original input columns are preserved.           |


## Quality Control (QC)

Each TCR entry is checked for basic validity:
-   **Gene names** – must be found in the species-specific V/J gene list.
-   **CDR3 length** – must be between **7 and 22 amino acids**.    
-   **Symbols** – only the 20 standard amino acids are allowed.
-   **Flank coherence** – the first and last residues of the CDR3 must match the expected V/J germline context.
   
QC issues are reported in the **`problem`** column.  
By default, sequences with QC failures are **skipped**. To override this behavior, use `--no-qc`.

---

## Examples

```bash
### 1) List embedded epitope models

# List to stdout
./TEMPO list_epitopes

# Save as CSV/TSV for browsing/filtering
./TEMPO list_epitopes --csv epitopes.csv
./TEMPO list_epitopes --tsv epitopes.tsv

---

### 2) Correct V/J gene names (map to IMGT names)

# Human αβ (default)
./TEMPO correct_VJ ./test/test_correction_VJ.csv ./test/clean_AB.csv --species HomoSapiens --chain AB

# Human α-only
./TEMPO correct_VJ ./test/test_correction_VJ.csv ./test/clean_A.csv --species HomoSapiens --chain A

# Human β-only
./TEMPO correct_VJ ./test/test_correction_VJ.csv ./test/clean_B.csv --species HomoSapiens --chain B

# Mouse αβ
./TEMPO correct_VJ ./test/test_mus.csv ./test/clean_mouse_AB.csv --species MusMusculus --chain AB

---

### 3) Predict with an embedded epitope model

# Human αβ (default chain = AB)
./TEMPO predict ./test/clean_AB.csv ./test/out.csv A0201_LLWNGPMAV

# Human α-only
./TEMPO predict ./test/clean_A.csv ./test/out_A.csv A0201_LLWNGPMAV --chain A

# Human β-only
./TEMPO predict ./test/clean_B.csv ./test/out_B.csv A0201_LLWNGPMAV --chain B

# Mouse αβ (example epitope model)
./TEMPO predict ./test/clean_mouse_AB.csv ./test/out_mouse.csv H2Kb_SIINFEKL --species MusMusculus

---

### 4) Include rows even if QC fails (use with care)
./TEMPO predict ./test/clean_B.csv ./test/out_B_noqc.csv A0201_LLWNGPMAV --chain B --no-qc

---

### 5) If you have a .bin motif file, use an explicit motif file instead of an embedded epitope ID
./TEMPO predict ./test/clean_AB.csv ./test/out_from_file.csv --motif ./motifs/HomoSapiens/A0201_LLWNGPMAV.bin

```

---

## Citation

> **XXX**  
> *TEMPO: motif‑based prediction of TCR–epitope recognition.*


## License

MIT for academic use. Commercial users: contact **[XX@XX.org](mailto:XX@XX.org)**.

