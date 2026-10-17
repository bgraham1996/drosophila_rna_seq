# Drosophila Brain RNA-seq — Context for Claude Desktop

> Pass this file to Claude Desktop to get up to speed on the project state.

---

## What this project is

Bulk RNA-seq analysis of *Drosophila melanogaster* brain tissue comparing three nutritional/behavioral conditions:

- **Starved** (4 samples: SRR29720968–SRR29720971)
- **Fed** (4 samples: SRR29720972–SRR29720975)
- **Untrained** — naive/unstimulated control (4 samples: SRR29720976–SRR29720979)

12 paired-end samples total, sourced from NCBI SRA (accessions SRR29720968–SRR29720979). The goal is to identify genes and biological processes differentially regulated by feeding state in the fly brain.

---

## Where everything lives

```
~/server/drosophila_brain_rna_seq_data_1/
```

### Scripts
| File | Purpose | Status |
|---|---|---|
| `trimming.py` | fastp QC trimming | Done |
| `alignment.py` | STAR genome index build | Done |
| `rna_alignment.py` | STAR read alignment | Done |
| `feature_counts.py` | featureCounts quantification | Done |
| `expression_changes_cleaning.py` | DESeq2 results post-processing | Done |
| `salmon_run.py` | Salmon quantification | **Empty — not started** |
| `test.py` | Scratch/diagnostic | N/A |

### Key data directories
| Directory / File | Contents |
|---|---|
| `fastq_files/` | Raw paired-end FASTQ (12 samples × 2 reads) |
| `trimmed_fastq/` | fastp-trimmed FASTQ |
| `genome_index/` | STAR genome index (BDGP6.54) |
| `bam_files/` | 12 coordinate-sorted BAM files + STAR logs |
| `feature_counts_output/counts.txt` | Raw gene-level count matrix |
| `R_notebooks/initial_dseq_analysis/` | DESeq2 R workspace + 3 results CSVs |
| `cleaned_expression_changes/` | Top 75 DEGs per comparison (padj-sorted) |
| `go_analysis/` | GO annotation files (not yet analysed) |
| `reference_genome/` | BDGP6.54 genome FASTA + GTF |
| `reference_transcriptome/` | BDGP6.54 cDNA FASTA |
| `reports/` | fastp HTML/JSON QC reports + MultiQC |

---

## Pipeline — what has been run

### 1. Quality trimming (fastp)
- Min length 36 bp, min quality Q20, paired-end mode
- Output: `trimmed_fastq/SRR*_trimmed_{1,2}.fastq.gz`
- MultiQC report: `reports/trimming/multiqc/multiqc_report.html`

### 2. Genome index (STAR)
- Reference: Ensembl BDGP6.54 (release 115)
- `--sjdbOverhang 150`, `--genomeSAindexNbases 12`, 4 threads
- Output: `genome_index/`

### 3. Read alignment (STAR)
- Paired-end, coordinate-sorted BAM, 8 threads, `zcat` for gzipped input
- Typical unique alignment rate: ~85% (e.g. SRR29720968: 85.44%)
- Multi-mapping: ~5–8% per sample
- Output: `bam_files/SRR*_Aligned.sortedByCoord.out.bam`
- **Note: BAI index files do not exist yet** — needed for genome browsers / some tools

### 4. Strandedness check (RSeQC)
- Result: library is **unstranded** (`-s 0`)
- Evidence: `strandedness_test_s0.txt` / `strandedness_test_s0.txt.summary`

### 5. Feature counting (featureCounts / subread)
- `-p --countReadPairs` (paired-end), `-s 0` (unstranded), 4 threads
- GTF: BDGP6.54.115
- Output: `feature_counts_output/counts.txt`

### 6. Differential expression (DESeq2 in R)
Three pairwise comparisons, all saved in `R_notebooks/initial_dseq_analysis/`:
- `res_starved_vs_fed.csv`
- `res_starved_vs_untrained.csv`
- `res_untrained_vs_fed.csv`

Columns: `baseMean`, `log2FoldChange`, `lfcSE`, `stat`, `pvalue`, `padj`
R workspace saved as `deseq2_analysis.RData`.

### 7. Results cleaning (`expression_changes_cleaning.py`)
- Drops rows with NA padj, sorts by padj ascending, keeps top 75 hits per comparison
- Output in `cleaned_expression_changes/`:
  - `starved_vs_fed_cleaned.csv`
  - `starved_vs_untrained_cleaned.csv`
  - `untrained_vs_fed_cleaned.csv`

**Top 5 DEGs — Starved vs Fed (by padj):**
| Gene ID | baseMean | log2FC | padj |
|---|---|---|---|
| FBgn0050467 | 250.7 | +6.37 | 2.23e-11 |
| FBgn0000552 | 58.5 | +19.83 | 9.82e-08 |
| FBgn0002872 | 154.3 | +5.98 | 3.13e-07 |
| FBgn0038769 | 97.0 | +6.03 | 4.24e-07 |
| FBti0020244 | 56.2 | -5.38 | 1.39e-05 |

---

## What is NOT done yet (next steps)

1. **GO enrichment analysis** — data files are downloaded but no script exists yet.
   - Annotation: `go_analysis/gene_association.fb` (FlyBase FB2026_01, Feb 2026)
   - Ontology: `go_analysis/go-basic.obo` (Gene Ontology Consortium)
   - Candidate tools: `goatools` (Python) or `clusterProfiler` (R)

2. **Salmon quantification** — `salmon_run.py` is empty; would give transcript-level estimates as an alternative/complement to the STAR→featureCounts counts.
   - Transcriptome available: `reference_transcriptome/Drosophila_melanogaster.BDGP6.54.cdna.all.fa`

3. **Visualizations** — no volcano plots, heatmaps, or PCA plots generated yet.

4. **BAM indexing** — no `.bai` files; needed before using genome browsers or tools like IGV.

---

## Environment

- Python 3.13+, managed with `uv` (`pyproject.toml`)
- Python dependencies: pandas, biopython, requests, python-dotenv, pathlib
- External bioinformatics tools: fastp, STAR, featureCounts (subread), multiqc
- R packages: DESeq2
- Project name in pyproject.toml: `drosophila-brain-rna-seq`
