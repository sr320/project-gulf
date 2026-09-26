# Analysis plan: SRP139854 (*C. virginica* gill MBD-BS, oil exposure)

BioProject: PRJNA449904 · SRA: SRP139854

## Data

| Run | Sample | Treatment | Reads | Gb |
|---|---|---|---|---|
| SRR6995997 | NB3 | No oil | 17.3 M | 1.75 |
| SRR6995998 | NB6 | No oil | 24.8 M | 2.50 |
| SRR6995995 | NB11 | No oil | 39.8 M | 4.02 |
| SRR6995996 | HB2 | 25,000 ppm oil | 33.4 M | 3.37 |
| SRR6995993 | HB16 | 25,000 ppm oil | 25.8 M | 2.60 |
| SRR6995994 | HB30 | 25,000 ppm oil | 9.2 M | 0.93 |

- Library: Bisulfite-Seq with "5-methylcytidine antibody" selection, so this is MBD-BS (methyl enrichment followed by bisulfite conversion), not WGBS.
- Single-end 101 bp reads, Illumina HiSeq 2500. Gill tissue, 2015.
- SRA has 6 runs (3 control, 3 oil). The local metadata has 13 rows with repeated samples; this still needs reconciling.
- HB30 has about a third of the depth of the other samples.

## Data quality findings (2026-09-25)
- **Adapter dimers survived default trimming.** Trim Galore's auto-detected 13 bp adapter missed TruSeq dimers with errors in the first bases or the first 4 bases missing: 1.6–6.5% of trimmed reads, 22% in HB30. Fixed in `02` by passing the full 33 bp adapter plus a 4 bp-truncated version.
- **Many reads are not bisulfite-converted.** Based on C/G content, about 17–31% of reads in five samples and about 63% in HB30 look unconverted. Before filtering, Bismark reported CHH methylation of 22–73% (should be <1%). `04` removes them with `filter_non_conversion` (≥3 methylated non-CpG calls).
- **Library looks PBAT-style.** 81–96% of alignments are on the complementary strands (CTOT/CTOB), not ~50%. Aligning with `--non_directional`.
- **Mapping is low.** 22–32% on 500k-read tests with the default `score_min`. `04` compares `L,0,-0.6`.

## Steps

### 1. Get the data
- Download with `prefetch` and `fasterq-dump` from SRA Toolkit, then gzip and record md5 checksums.
- Use `runinfo.csv` as the sample sheet and add a `treatment` column.

### 2. QC and trimming
- Run FastQC and MultiQC on the raw reads.
- Trim with Trim Galore or fastp for adapters and quality. After the first alignment, check M-bias plots and clip biased read ends if needed.

### 3. Reference genome
- *C. virginica* C_virginica-3.0 (GCF_002022765.2): genome FASTA plus NCBI GFF.
- Build the index with `bismark_genome_preparation` (Bowtie2).

### 4. Alignment
- Run Bismark in single-end mode.
- Decision: is the library directional? Zymo Pico Methyl-Seq needs `--non_directional`.
- Decision: whether to deduplicate. MBD enrichment plus single-end reads means some duplicates may be real. Run with and without deduplication and compare.
- Record mapping rate and bisulfite conversion efficiency (from non-CpG methylation) for each sample.

### 5. Methylation calling
- Run `bismark_methylation_extractor` and produce CpG coverage reports.
- Merge the two strands of each CpG (`coverage2cytosine --merge_CpG`).
- Summarize coverage for each sample and decide whether HB30 is usable.

### 6. Differential methylation (oil vs. control)
- DMLs: methylKit, at least 5–10× coverage in every sample, logistic regression (3 v 3), q < 0.01, Δ ≥ 25%.
- DMRs: methylKit tiles, or DSS (better with small sample sizes).
- PCA and hierarchical clustering: do samples group by treatment, and is HB30 an outlier?
- Caveat: MBD enrichment favors methylated regions, so results only apply to the enriched fraction. Coverage differences may themselves carry signal.

### 7. Annotation and function
- Build feature tracks from the GFF: gene bodies, exons, introns, promoters (about 1 kb upstream), intergenic regions, and TEs if a track is available.
- Use `bedtools intersect` to place DMLs and DMRs in those features, and compare against all covered CpGs.
- GO enrichment with topGO or GO-MWU, using genes with coverage as the background.

### 8. Reproducibility
- Repo layout: `data/`, `code/` (numbered Quarto/Rmd files), `output/`, `envs/` (conda YAML).
- Run Bismark on HPC. Total input is about 15 Gb, so allow a few CPU-hours per sample.

## Open questions
1. Which library kit was used? The strand pattern suggests PBAT-style. Why were so many reads (most of HB30) not bisulfite-converted: incomplete conversion, or unconverted DNA carried through MBD enrichment?
2. What are the extra rows in the local metadata: lanes, re-sequencing, or duplicates? Is there data not deposited in SRA, especially for HB30?
3. Is there a linked publication or prior analysis to reproduce or extend?
