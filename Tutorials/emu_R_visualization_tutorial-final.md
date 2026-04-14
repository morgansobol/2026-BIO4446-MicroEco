# Visualizing EMU Results in R
### A follow-up to the EMU 16S tutorial

---

This tutorial uses two files you already have:

| File | Description |
|------|-------------|
| `emu-combined-species.tsv` | EMU combined output — 336 species × 8 samples, relative abundance |
| `metadata.csv` | Sample metadata — `Sample` and `Type` columns |

The EMU file has taxonomy columns (`species`, `genus`, `family`, `order`, `class`, `phylum`, `superkingdom`) followed by one column per sample (`Rock_1`, `Rock_2`, `Water_1`, `Water_2`, `Plant_1`, `Plant_2`, `Sediment_1`, `Sediment_2`). Absent taxa are `NA` rather than `0` — we handle that on import.

---

## Table of Contents

1. [Install and Load Packages](#1-install-and-load-packages)
2. [Load Your Data](#2-load-your-data)
3. [Build a Phyloseq Object](#3-build-a-phyloseq-object)
4. [Relative Abundance Bar Chart](#4-relative-abundance-bar-chart)
5. [Alpha Diversity](#5-alpha-diversity)
6. [Beta Diversity (PCoA)](#6-beta-diversity-pcoa)
7. [Differential Abundance with ANCOMBC2](#7-differential-abundance-with-ancombc2)
8. [Save All Plots](#8-save-all-plots)

---

## 1. Install and Load Packages

Run this section **once** to install everything needed.

```r
# CRAN packages
install.packages(c(
  "tidyverse",
  "vegan",
  "patchwork",
  "RColorBrewer"
))

# Bioconductor packages
if (!require("BiocManager", quietly = TRUE))
    install.packages("BiocManager")

BiocManager::install(c(
  "phyloseq",
  "ANCOMBC"
))
```

Load for each session:

```r
library(tidyverse)
library(phyloseq)
library(vegan)
library(patchwork)
library(RColorBrewer)
library(ANCOMBC)
```

---

## 2. Load Your Data

```r
# Load the EMU combined output
emu <- read_tsv("emu-combined-species.tsv")

# Load metadata
meta <- read_csv("metadata.csv")

head(emu)
# species | genus | family | ... | Rock_1 | Water_1 | Plant_2 | ...

head(meta)
# Sample     | Type
# Water_1    | Water
# Plant_1    | Plant
# Sediment_1 | Sediment
# Rock_1     | Rock
```

Define sample names for convenience:

```r
sample_cols <- c("Rock_1", "Rock_2", "Water_1", "Water_2",
                 "Plant_1", "Plant_2", "Sediment_1", "Sediment_2")
```

---

## 3. Build a Phyloseq Object

Phyloseq keeps your abundance table, taxonomy, and metadata in one tidy object for analysis and plotting.

### 3.1 OTU matrix

NAs in the EMU file mean a taxon was absent in that sample. Replace with 0.

```r
otu_mat <- emu %>%
  select(species, all_of(sample_cols)) %>%
  mutate(across(all_of(sample_cols), ~replace_na(.x, 0))) %>%
  column_to_rownames("species") %>%
  as.matrix()

dim(otu_mat)  # 336 taxa x 8 samples
```

### 3.2 Taxonomy table

```r
tax_mat <- emu %>%
  select(species, genus, family, order, class, phylum, superkingdom) %>%
  column_to_rownames("species") %>%
  as.matrix()
```

### 3.3 Sample metadata

The metadata `Sample` column matches the EMU column names exactly (`Rock_1`, `Water_1`, etc.).

```r
meta_df <- meta %>%
  column_to_rownames("Sample")

# Confirm names match
all(rownames(meta_df) %in% colnames(otu_mat))  # should be TRUE
```

### 3.4 Assemble

```r
ps <- phyloseq(
  otu_table(otu_mat, taxa_are_rows = TRUE),
  tax_table(tax_mat),
  sample_data(meta_df)
)

ps
# phyloseq-class experiment-level object
# otu_table()   OTU Table:      [ 336 taxa and 8 samples ]
# sample_data() Sample Data:    [ 8 samples by 1 variable ]
# tax_table()   Taxonomy Table: [ 336 taxa by 7 taxonomic ranks ]
```

---

## 4. Relative Abundance Bar Chart

### 4.1 Aggregate to phylum level

There are 19 phyla in this dataset. We collapse anything below 1% mean abundance into "Other" to keep the plot readable.

```r
ps_phylum <- tax_glom(ps, taxrank = "phylum")
ps_rel    <- transform_sample_counts(ps_phylum, function(x) x / sum(x))

df_bar <- psmelt(ps_rel) %>%
  rename(Phylum = phylum) %>%
  mutate(Phylum = ifelse(Abundance < 0.01, "Other (< 1%)", Phylum))
```

### 4.2 Plot

```r
# Manually assign colours to the dominant phyla in this dataset
pal <- c(
  "Proteobacteria"  = "#4E79A7",
  "Cyanobacteria"   = "#59A14F",
  "Bacteroidetes"   = "#F28E2B",
  "Firmicutes"      = "#E15759",
  "Acidobacteria"   = "#B07AA1",
  "Other (< 1%)"    = "#BAB0AC"
)
# Catch any remaining phyla automatically
extra_phyla <- setdiff(unique(df_bar$Phylum), names(pal))
extra_cols  <- setNames(
  colorRampPalette(brewer.pal(8, "Set2"))(length(extra_phyla)),
  extra_phyla
)
pal <- c(pal, extra_cols)

p_bar <- ggplot(df_bar, aes(x = Sample, y = Abundance, fill = Phylum)) +
  geom_bar(stat = "identity", color = "white", linewidth = 0.2) +
  facet_grid(~ Type, scales = "free_x", space = "free_x") +
  scale_fill_manual(values = pal) +
  scale_y_continuous(labels = scales::percent_format()) +
  labs(
    title    = "Community composition by phylum",
    subtitle = "Relative abundance, grouped by environment type",
    x = NULL, y = "Relative Abundance", fill = "Phylum"
  ) +
  theme_bw(base_size = 12) +
  theme(
    axis.text.x     = element_text(angle = 45, hjust = 1),
    strip.text      = element_text(face = "bold"),
    legend.position = "right"
  )

p_bar
```

> **What to expect:** Plant samples are Cyanobacteria-dominated (phototrophic biofilm communities). Rock samples are extremely sparse — Rock_1 has only 6 detected species, almost certainly due to low read depth. Sediment_2 is exceptionally diverse (245 species vs. 50 in Sediment_1).

---

## 5. Alpha Diversity

Alpha diversity measures richness and evenness **within** each sample.

> **Note on input format:** phyloseq's `estimate_richness()` expects integer read counts, but EMU outputs relative abundances. We scale up by 10,000 and round to create pseudo-counts. This preserves the rank order of diversity while satisfying phyloseq's requirements.

```r
ps_counts <- transform_sample_counts(ps, function(x) round(x * 10000))

alpha_df <- estimate_richness(ps_counts, measures = c("Observed", "Shannon")) %>%
  rownames_to_column("Sample") %>%
  left_join(meta, by = "Sample")

print(alpha_df)
```

Expected values from this dataset:

| Sample | Shannon | Observed |
|--------|---------|----------|
| Rock_1 | 2.6 | 6 |
| Rock_2 | 4.4 | 25 |
| Water_1 | 5.4 | 53 |
| Water_2 | 5.2 | 51 |
| Plant_1 | 4.8 | 72 |
| Plant_2 | 4.1 | 48 |
| Sediment_1 | 5.3 | 50 |
| Sediment_2 | 6.7 | 245 |

### 5.1 Shannon diversity plot

```r
p_shannon <- ggplot(alpha_df, aes(x = Type, y = Shannon, color = Type)) +
  geom_jitter(size = 4, width = 0.1, alpha = 0.9) +
  scale_color_brewer(palette = "Set1") +
  labs(
    title    = "Alpha Diversity — Shannon Index",
    subtitle = "Two replicates per environment type",
    x = NULL, y = "Shannon Diversity (bits)"
  ) +
  theme_bw(base_size = 12) +
  theme(legend.position = "none")

p_shannon
```

### 5.2 All metrics side by side

```r
alpha_long <- alpha_df %>%
  pivot_longer(cols = c(Observed, Shannon), names_to = "Metric", values_to = "Value")

p_alpha_all <- ggplot(alpha_long, aes(x = Type, y = Value, color = Type)) +
  geom_jitter(size = 3, width = 0.1, alpha = 0.85) +
  facet_wrap(~ Metric, scales = "free_y") +
  scale_color_brewer(palette = "Set1") +
  labs(title = "Alpha Diversity", x = NULL, y = "Value") +
  theme_bw(base_size = 11) +
  theme(axis.text.x = element_text(angle = 35, hjust = 1), legend.position = "none")

p_alpha_all
```

---

## 6. Beta Diversity (PCoA)

Beta diversity shows how **different** communities are between samples. Bray-Curtis ranges from 0 (identical) to 1 (completely different).

```r
bray_dist <- phyloseq::distance(ps, method = "bray")
ord       <- ordinate(ps, method = "PCoA", distance = bray_dist)
```

### 6.1 PCoA plot

```r
p_pcoa <- plot_ordination(ps, ord, color = "Type", label = "Sample") +
  geom_point(size = 5, alpha = 0.9) +
  scale_color_brewer(palette = "Set1") +
  labs(
    title    = "Beta Diversity — PCoA (Bray-Curtis)",
    subtitle = "Closer points = more similar communities",
    color    = "Environment Type"
  ) +
  theme_bw(base_size = 12)

p_pcoa
```

> **What to expect:**
> - Plant replicates cluster tightly (Bray-Curtis ~0.18) — consistent biofilm community
> - Water replicates are moderately similar (~0.39)
> - Sediment replicates are more spread apart (~0.47), driven by Sediment_2's unusually high diversity
> - Rock replicates are completely dissimilar (BC = 1.0) — Rock_1 and Rock_2 share zero species, almost certainly a low-read-depth artefact in Rock_1

### 6.2 PERMANOVA — does environment type explain community composition?

```r
otu_for_perm <- t(as(otu_table(ps), "matrix"))
meta_aligned  <- as(sample_data(ps), "data.frame")

set.seed(42)
permanova_result <- adonis2(
  otu_for_perm ~ Type,
  data         = meta_aligned,
  method       = "bray",
  permutations = 999
)

print(permanova_result)
```

> **Caution:** With only 2 replicates per group (8 total), PERMANOVA has very low statistical power. A significant result is still informative, but a non-significant result should not be taken as evidence that environment types have the same community.

---

## 7. Differential Abundance with ANCOMBC2

ANCOMBC2 identifies taxa that differ significantly between groups, correcting for compositionality bias. We run it at genus level to increase statistical power.

```r
# Scale relative abundances to pseudo-counts for ANCOMBC2
ps_counts <- transform_sample_counts(ps, function(x) round(x * 10000))

set.seed(42)
ancom_result <- ancombc2(
  data         = ps_counts,
  tax_level    = "genus",
  fix_formula  = "Type",
  p_adj_method = "BH",
  verbose      = FALSE
)

res_df <- ancom_result$res

# Show genera significant in at least one comparison (q < 0.05)
sig_genera <- res_df %>%
  filter(if_any(starts_with("q_"), ~ . < 0.05))

cat("Significant genera found:", nrow(sig_genera), "\n")
```

### Visualise log-fold changes

The reference group is the first level alphabetically — `Plant` by default. All comparisons are relative to Plant.

```r
lfc_long <- res_df %>%
  filter(taxon %in% sig_genera$taxon) %>%
  select(taxon, starts_with("lfc_")) %>%
  pivot_longer(
    cols      = starts_with("lfc_"),
    names_to  = "comparison",
    values_to = "lfc"
  ) %>%
  mutate(comparison = str_remove(comparison, "lfc_Type"))

p_ancom <- ggplot(lfc_long, aes(x = lfc, y = reorder(taxon, lfc), fill = comparison)) +
  geom_col(position = "dodge") +
  geom_vline(xintercept = 0, linetype = "dashed", color = "grey40") +
  scale_fill_brewer(palette = "Set1") +
  labs(
    title    = "Differential Abundance (ANCOMBC2)",
    subtitle = "Log-fold change vs. Plant (reference); q < 0.05",
    x = "Log-Fold Change", y = "Genus", fill = "vs. Plant"
  ) +
  theme_bw(base_size = 11)

p_ancom
```

> With only 8 samples, expect few significant hits. Treat trends in LFC as hypothesis-generating rather than confirmatory.

---

## 8. Save All Plots

```r
ggsave("bar_chart_phylum.pdf",       p_bar,       width = 10, height = 5)
ggsave("alpha_diversity.pdf",        p_alpha_all, width = 8,  height = 4)
ggsave("beta_diversity_pcoa.pdf",    p_pcoa,      width = 7,  height = 5)
ggsave("differential_abundance.pdf", p_ancom,     width = 9,  height = 5)

# Or combine into one multi-panel figure
combined <- (p_bar) /
            (p_alpha_all | p_pcoa) +
  plot_annotation(
    title   = "Environmental Microbiome — EMU 16S Analysis",
    caption = "8 samples: Plant, Rock, Water, Sediment (2 replicates each)"
  )

ggsave("combined_figure.pdf", combined, width = 12, height = 14)
```

---

## Data notes for interpretation

| Observation | Likely explanation |
|-------------|-------------------|
| Rock_1 has only 6 detected species | Very low read depth — interpret with caution |
| Rock_1 and Rock_2 share no species (BC = 1.0) | Consequence of low reads in Rock_1 |
| Sediment_2 has 245 species vs. 50 in Sediment_1 | Genuine biological heterogeneity or sequencing depth difference |
| Plant replicates cluster tightly (BC = 0.18) | Consistent Cyanobacteria-dominated biofilm community |
| Dominant phyla: Proteobacteria + Cyanobacteria | Expected for phototrophic environmental surface samples |

---

*Validated with `emu-combined-species.tsv` (336 species, 8 samples) and `metadata.csv`.*
