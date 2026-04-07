# 16S Taxonomic Profiling with EMU
### A tutorial for Nanopore full-length 16S amplicon data

---
#### Emu is a relative abundance estimator for 16S genomic sequences. The method is optimized for error-prone full-length reads, but can also be utilized for short-read data.

<img width="1520" height="736" alt="image" src="https://github.com/user-attachments/assets/a1787e46-da57-436d-bc46-d59146a4e7dd" />

Emu was given its name because it uses an expectation-maximization (EM) approach for microscopic (mu, 𝛍) organisms, yielding EM + mu or Emu for short. 

This algorithm is constructed upon the idea that an unclassified sequence is more likely to come from an organism that is expected to be in the sample at high abundance rather than an organism that is either detected in low abundance or not at all. 

Emu first produces read classifications by mapping reads to a database of 16S sequences and a direct proportion community profile that is expected to have some inaccuracies due to error-prone sequences. The initial guess profile is then used to update the read classification likelihood giving more weight to higher abundance species. These updated read classification likelihoods then update the community profile directly. This process continues until only marginal changes are made to the community profile between iterations. A clean-up step is completed to remove low likelihood species and the final community profile estimation is ultimately returned.

---

## Table of Contents

1. [Directory Setup](#1-directory-setup)
2. [Installation](#2-installation)
3. [Running EMU](#3-running-emu)
4. [Understanding the Output](#4-understanding-the-output)
5. [Visualizing in R — Community Composition](#5-visualizing-in-r--community-composition)
6. [Alpha Diversity](#6-alpha-diversity)
7. [Beta Diversity (Ordination)](#7-beta-diversity-ordination)

---
## 1. Directory Setup

Organise your project before running EMU. 

### Expected structure

```
2026-MicroEco16S/
├── data/
│   ├── Plant_1.fastq
│   ├── Plant_2.fastq
│   ├── Rock_1.fastq
│   ├── Rock_2.fastq
│   ├── Sediment_1.fastq
│   ├── Sediment_2.fastq
│   ├── Water_1.fastq
│   └── Water_2.fastq
├── results/     
└── figures/      
└── db/      

```
### Create database folder and output directories

You should be in the `2026-MicroEco16S` directory, if not move there with `cd`. 

Make new directories to hold our database and to output figures. Emu will create the `results/` directory for us. 

```bash
mkdir -p db figures
```
```bash
ls
```

Get the `data/` directory from Canvas. Choose to download it to `2026-MicroEco16S` if you can. If it goes automatically to `Downloads` don't fret, we can move it here, like so:
```bash
mv ../../Downloads/data.zip .
```
With this command, I moved the files from a few directories back and placed it in our current location (i.e. `2026-MicroEco16S`) using the `.` to signify "here".

Now `unzip` the zip file.
```bash
unzip data.zip
```
```bash
ls data/
```
You should now see your fastq files in `data/`. 

Let's take a look at one using the `head` command.
```bash
head data/Plant_1.fastq
```

The fastq format has 4 lines per sequence: 
* the sequence identifier (header), preceded by a “@” character;
* the nucleic acid sequence itself;
* a “+” character and possibly the header information repeated;
* and the quality score information for each individual basecall, which must contain the same number of characters as letters in the sequence. 

![Breakdown of a fastq file](https://github.com/user-attachments/assets/e7a64482-ccae-4ee4-99a8-4bb83141b448)

### Now, let's check read counts before running.
We will use the command `grep` which is a command-line utility for searching text for lines that match a specific character or string of text. Here, we will search for "@" which signifies a new sequence in the file. 
```bash
grep -c '^@' data/*.fastq
```

> **Low-read samples:** If any samples have fewer than ~100 reads (e.g. Rock samples), EMU will still run but results will be unreliable. Flag these in your analysis and interpret with caution.


## 2. Installation of Emu

EMU runs on macOS and Linux. To install it we will use the package manager Conda. It is the recommended package installer for most workflows. It installs all packages and dependecies. Conda itself is already installed on your computers. 

### Conda

```bash
conda create -n emu -c bioconda -c conda-forge emu python=3.7
```

It will ask you to `Proceed ([y]/n)?`, press `y` and `enter`

Now activate the environment to be able to use the program `emu`. 
```bash
conda activate emu
```

# Verify install
```
emu --version
```

### Download the EMU database using osf client

Now we need to download a separate package using `pip` that will help us download the database. 

```bash
pip install osfclient
```

We want to download the database in the `db/` folder, so move there first. 
```bash
cd db/
```
```bash
osf -p 32sh5 fetch osfstorage/species_taxid.fasta
```
```bash
osf -p 32sh5 fetch osfstorage/taxonomy.tsv
```

Check the files are in your `db` directory
```bash
ls
```

Go back one directory to main `2026-MicroEco16S/` directory
```bash
cd ../
```

---

## 3. Running EMU

EMU aligns reads against its 16S database using minimap2 and estimates taxon abundances. 

### Run all samples at once using a `for` loop
A `for` loop is a programming control structure used to execute a block of code repeatedly. Here, we are telling the computer: "For every .fastq file in the data/ folder, run the emu abundance command on it."
We are creating a variable called `sample` to store the name of each file, without the folder path or the .fastq ending.
For example, if `$f` is data/Plant_1.fastq, then:

`basename` strips the folder path → Plant_1.fastq
`.fastq` tells it to also remove the extension → Plant_1
So `sample` becomes Plant_1

We then use `$sample` to name the output folder: results/Plant_1, results/Plant_2, etc. — keeping each sample's results neatly separated.

Then we run emu on each sample, provide the path to the database, and set threads=6 (i.e. to use 6 computer nodes). 

```bash
for f in data/*.fastq; do
  # Extract sample name (e.g. Plant_1)
  sample=$(basename "$f" .fastq)

  emu abundance "$f" --db db/ --threads 6
done
```

### Key parameters

| Flag | Value | Notes |
|------|-------|-------|
| `--type` | `map-ont` | Nanopore preset for minimap2. Do not use `map-pb` or `sr`. |
| `--db` | path/to/db | Path to downloaded EMU database directory. |
| `--threads` | 4 | MacBook Air has limited cores — 4 is safe. |
| `--min-abundance` | 0.0001 | Optional. Filters taxa below 0.01% relative abundance. |
| `--keep-files` | (flag) | Optional. Keeps intermediate sam/bam files for QC inspection. |

> **Runtime estimate:** Each sample takes roughly 2–5 minutes on a MacBook Air with 4 threads, depending on read count.

### Combine all samples into one table

```bash
# Merge abundance tables from all samples (raw counts)
emu combine-outputs results/ --output results/combined_counts.tsv

# Also create a relative abundance version
emu combine-outputs results/ --output results/combined_rel.tsv --rel-abund
```

---

## 4. Understanding the Output

EMU produces a tab-separated abundance table for each sample. After combining, you get a merged matrix ready for R.

### Per-sample output (`results/Plant_1/`)

```
tax_id    abundance    species                    genus           family
1234567   0.3421       Pseudomonas fluorescens    Pseudomonas     Pseudomonadaceae
2345678   0.1892       Bacillus subtilis          Bacillus        Bacillaceae
3456789   0.0934       Sphingomonas sp.           Sphingomonas    Sphingomonadaceae
...
```

### Combined output (`results/combined_rel.tsv`)

```
tax_id    species                      Plant_1    Plant_2    Rock_1    ...
1234567   Pseudomonas fluorescens      0.342      0.298      0.000     ...
2345678   Bacillus subtilis            0.189      0.201      0.412     ...
...
```

### Output file index

| File | Contents |
|------|----------|
| `*_emu_abundance.tsv` | Per-sample relative abundances with full taxonomy |
| `combined_counts.tsv` | Raw read counts, all samples merged |
| `combined_rel.tsv` | Relative abundances, all samples merged — use this for R |
| `*.sam` / `*.bam` | Alignment files (only with `--keep-files`). Useful for QC. |

> **Unassigned reads:** EMU reports reads that couldn't be assigned as "unclassified". A high unclassified fraction (>30%) may indicate contamination, poor DNA quality, or organisms not well-represented in the database.

---

## 5. Visualizing in R — Community Composition

Load the combined EMU output into R and create stacked bar charts of community composition. This is the standard first visualization for 16S data.

### Load and reshape

```r
library(tidyverse)
library(ggplot2)

# Load combined relative abundance table
emu <- read_tsv("results/combined_rel.tsv")

# Reshape to long format for ggplot
emu_long <- emu |>
  pivot_longer(
    cols = -c(tax_id, species, genus, family, order, class, phylum),
    names_to  = "sample",
    values_to = "rel_abund"
  )
```

### Stacked bar chart at phylum level

```r
# Collapse to phylum, group rare taxa as "Other"
phylum_plot <- emu_long |>
  group_by(sample, phylum) |>
  summarise(rel_abund = sum(rel_abund), .groups = "drop") |>
  mutate(phylum = ifelse(rel_abund < 0.02, "Other (<2%)", phylum))

# Plot
ggplot(phylum_plot, aes(x = sample, y = rel_abund, fill = phylum)) +
  geom_col() +
  scale_y_continuous(labels = scales::percent) +
  theme_bw() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
  labs(x = "", y = "Relative abundance", fill = "Phylum")

ggsave("figures/phylum_barplot.pdf", width = 8, height = 5)
```

---

## 6. Alpha Diversity

**Alpha diversity** measures richness and evenness *within* each sample. The two most common metrics for 16S data are the Shannon index (accounts for evenness) and the Simpson index (dominance-based).

### Interpreting Shannon index

| Value | Interpretation |
|-------|----------------|
| 0 | One taxon dominates completely |
| 1–2 | Low diversity |
| 3–4 | Typical for diverse environmental samples |
| >4 | Very high diversity |

> Low Shannon values (<1) may indicate genuinely low diversity or insufficient reads — check your read counts first.

### Calculate in R

```r
library(vegan)

# Convert to wide matrix (samples as rows, species as columns)
otu_mat <- emu |>
  column_to_rownames("species") |>
  select(where(is.numeric)) |>
  t()

# Shannon and Simpson diversity
alpha <- data.frame(
  sample   = rownames(otu_mat),
  shannon  = diversity(otu_mat, index = "shannon"),
  simpson  = diversity(otu_mat, index = "simpson"),
  richness = specnumber(otu_mat)
)

# Add sample type (Plant, Rock, Sediment, Water)
alpha <- alpha |>
  mutate(type = str_extract(sample, "[A-Za-z]+"))
```

### Plot alpha diversity

```r
ggplot(alpha, aes(x = type, y = shannon, colour = type)) +
  geom_jitter(width = 0.1, size = 3) +
  stat_summary(fun = mean, geom = "crossbar", width = 0.4) +
  theme_bw() +
  labs(y = "Shannon index", x = "Sample type", colour = "")

ggsave("figures/alpha_diversity.pdf", width = 6, height = 4)
```

---

## 7. Beta Diversity (Ordination)

**Beta diversity** compares community composition *between* samples. PCoA (Principal Coordinates Analysis) on Bray-Curtis dissimilarity is the standard approach — it shows which samples have similar microbial communities.

### Calculate Bray-Curtis dissimilarity and run PCoA

```r
library(vegan)

# Bray-Curtis dissimilarity matrix
bray <- vegdist(otu_mat, method = "bray")

# PCoA ordination
pcoa <- cmdscale(bray, k = 2, eig = TRUE)

# Extract variance explained by each axis
pct <- round(pcoa$eig / sum(pcoa$eig[pcoa$eig > 0]) * 100, 1)

# Build data frame for plotting
pcoa_df <- data.frame(
  PC1    = pcoa$points[, 1],
  PC2    = pcoa$points[, 2],
  sample = rownames(otu_mat)
) |> mutate(type = str_extract(sample, "[A-Za-z]+"))
```

### PCoA plot

```r
library(ggrepel)

ggplot(pcoa_df, aes(x = PC1, y = PC2, colour = type, label = sample)) +
  geom_point(size = 4) +
  geom_label_repel(size = 3) +
  theme_bw() +
  labs(
    x      = paste0("PC1 (", pct[1], "% variance)"),
    y      = paste0("PC2 (", pct[2], "% variance)"),
    colour = ""
  )

ggsave("figures/beta_pcoa.pdf", width = 6, height = 5)
```

### What to look for

Biological replicates (`_1` and `_2`) of the same environment should cluster together on the plot. Samples from different environments (Plant vs Water vs Sediment) should separate along PC1 or PC2. If replicates don't cluster, check your DNA extraction and sequencing protocol for consistency issues.

### Test significance with PERMANOVA

PERMANOVA tests whether sample type statistically explains differences in community composition.

```r
# Metadata table
meta <- data.frame(
  sample = rownames(otu_mat),
  type   = str_extract(rownames(otu_mat), "[A-Za-z]+")
)

# PERMANOVA (999 permutations)
adonis2(bray ~ type, data = meta, permutations = 999)
```

A significant p-value (< 0.05) means community composition differs significantly between your sample types.

---

## Quick Reference — EMU vs DADA2

| | DADA2 | EMU |
|--|-------|-----|
| Designed for | Illumina short reads | Nanopore long reads |
| Error model | Quality score-based | Alignment-based |
| Typical read retention | ~3% on Nanopore | ~80–95% |
| Output | ASVs | Species-level abundances |
| Database required | No (denoise-first) | Yes (~3–8GB) |
| Best for | Short amplicons, Illumina | Full-length 16S, Nanopore |

---

*Tutorial prepared for 2026-MicroEco16S course. Sample data: Plant, Rock, Sediment, Water environments with biological replicates.*
