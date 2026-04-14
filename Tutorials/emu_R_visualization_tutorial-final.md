# Visualizing EMU Results in R
### A follow-up to the EMU 16S tutorial

---

This tutorial uses two files you should have:

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

```

Load for each session:

```r
library(tidyverse)
library(phyloseq)
library(vegan)
library(patchwork)
library(RColorBrewer)
```

---

## 2. Load Your Data

First need to set your working directory to `2026-MicroEco`older we created last week. 
```r
setwd("/user/student/Desktop/2026-MicroEco16S/")
```

```r
# Load Data ---------------------------------------------------------------

# Load the EMU combined output
emu <- read_tsv("./results-og/emu-combined-species.tsv")

# Load metadata
meta <- read_csv("metadata.csv")

# See correct import of files
head(emu)

head(meta)
```
Define your sample names as a list for convenience in plotting

```R
sample_cols <- c("Rock_1", "Rock_2", "Water_1", "Water_2",
                 "Plant_1", "Plant_2", "Sediment_1", "Sediment_2")
```

Clean up data, remove blanks and NAs
```R
emu_clean <- emu %>%
  filter(!is.na(species)) %>%
  mutate(
    genus  = if_else(is.na(genus),  paste0("unclassified_", family), genus),
    family = if_else(is.na(family), paste0("unclassified_", order),  family),
    order  = if_else(is.na(order),  paste0("unclassified_", class),  order),
    class  = if_else(is.na(class),  paste0("unclassified_", phylum), class)
  )
```

Define sample names for convenience:

```r
sample_cols <- c("Rock_1", "Rock_2", "Water_1", "Water_2",
                 "Plant_1", "Plant_2", "Sediment_1", "Sediment_2")
```

---

## 3. Build a Phyloseq Object

Phyloseq is an R package to import, store, analyze, and graphically display complex phylogenetic sequencing data that has already been clustered into OTUs or ASVs. It leverages and builds upon many of the tools available in R for ecology and phylogenetic analysis (vegan, ade, ape), while also using advanced/flexible graphic systems (ggplot2) to easily produce publication-quality graphics. Check out more, including tutorials on what else it can do: https://joey711.github.io/phyloseq/

Phyloseq keeps your abundance table, taxonomy, and metadata in one tidy object for analysis and plotting.

### 3.1 OTU matrix

```r
otu_mat <- emu_clean %>%
  select(species, all_of(sample_cols)) %>%
  mutate(across(all_of(sample_cols), ~replace_na(.x, 0))) %>%
  column_to_rownames("species") %>%
  as.matrix()

dim(otu_mat)  # 336 taxa x 8 samples
```

### 3.2 Taxonomy table

```r
tax_mat <- emu_clean %>%
  select(species, genus, family, order, class, phylum, superkingdom) %>%
  column_to_rownames("species") %>%
  as.matrix()
```

### 3.3 Sample metadata

The metadata `Sample` column matches the EMU column names exactly (`Rock_1`, `Water_1`, etc.).

```r
meta_df <- meta %>%
  column_to_rownames("Sample") %>%
  mutate(Sample = rownames(.))    

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
```

---

## 4. Relative Abundance Bar Chart

### 4.1 Aggregate to phylum level


```r
ps_phylum <- tax_glom(ps, taxrank = "phylum")
ps_rel    <- transform_sample_counts(ps_phylum, function(x) x / sum(x))

df_bar <- psmelt(ps_rel) %>%
  rename(Phylum = phylum)  
```

### 4.2 Plot

```r
pal <- c(
  "Proteobacteria"     = "#4E79A7",
  "Cyanobacteria"      = "#59A14F",
  "Bacteroidetes"      = "#F28E2B",
  "Firmicutes"         = "#E15759",
  "Acidobacteria"      = "#B07AA1",
  "Nitrospirae"        = "#76B7B2",
  "Actinobacteria"     = "#EDC948",
  "Ignavibacteriae"    = "#FF9DA7",
  "Planctomycetes"     = "#9C755F",
  "Chloroflexi"        = "#BAB0AC",
  "Verrucomicrobia"    = "#D4A6C8",
  "Spirochaetes"       = "#FFBE7D",
  "Armatimonadetes"    = "#A0CBE8",
  "Elusimicrobia"      = "#86BCB6",
  "Calditrichaeota"    = "#F1CE63",
  "Gemmatimonadetes"   = "#D7B5A6",
  "Fibrobacteres"      = "#C8A0D4",
  "Kiritimatiellaeota" = "#B6D36B",
  "Thermotogae"        = "#FA8775"
)

p_bar <- ggplot(df_bar, aes(x = Sample, y = Abundance, fill = Phylum)) +
  geom_bar(stat = "identity", color = NA) +
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
ggsave("figures/bar_chart_phylum.png",p_bar, width = 10, height = 5,  dpi = 300)
```

> **What to expect:** Plant samples are Cyanobacteria-dominated (phototrophic biofilm communities). Rock samples are extremely sparse — Rock_1 has only 6 detected species, almost certainly due to low read depth. Sediment_2 is exceptionally diverse (245 species vs. 50 in Sediment_1).

---

## 5. Alpha Diversity

**Alpha diversity** describes the diversity within a single community/sample. It considers the number of different species in that sample (also referred to as species richness). Additionally, it can take the abundance of each species into account to measure how evenly taxa are distributed across the sample (also referred to as species evenness). 

* Shannon's = Measures _diversity_ as both richness and evenness (relative proportions of our ASVs). The value increases as you add more taxa, even if they are rare.
* Simpson's = also measures both richness and evenness, but weights more on dominant taxa and is less sensitive to rare ones. 1 = very even; close to 0 = one or a few taxa dominate.

To compare alpha diversity across samples, would be to ask if the mean or median of these calculated indices differs across groups.

> [WARNING!]
> These are just some metrics to help compare & contrast our samples within an experiment, and should **not** be considered “true” values of any ASV.


> **Note on input format:** phyloseq's `estimate_richness()` expects integer read counts, but EMU outputs relative abundances. We scale up by 10,000 and round to create pseudo-counts. This preserves the rank order of diversity while satisfying phyloseq's requirements.

```r
ps_counts <- transform_sample_counts(ps, function(x) round(x * 10000))

alpha_df <- estimate_richness(ps_counts, measures = c("Observed", "Shannon", "Simpson")) %>%
  rownames_to_column("Sample") %>%
  left_join(meta, by = "Sample")

print(alpha_df)

```

### 5.1 Plot all metrics side by side

```r
alpha_long <- alpha_df %>%
  pivot_longer(cols = c(Observed, Shannon, Simpson), names_to = "Metric", values_to = "Value")

p_alpha_all <- ggplot(alpha_long, aes(x = Type, y = Value, color = Type)) +
  geom_jitter(size = 3, width = 0.1, alpha = 0.85) +
  facet_wrap(~ Metric, scales = "free_y") +
  scale_color_brewer(palette = "Set1") +
  labs(title = "Alpha Diversity", x = NULL, y = "Value") +
  theme_bw(base_size = 11) +
  theme(axis.text.x = element_text(angle = 35, hjust = 1), legend.position = "none")

p_alpha_all
ggsave("figures/alpha_diversity.png", p_alpha_all,  width = 10, height = 6,  dpi = 300)
```

---

## 6. Beta Diversity (PCoA)

**Beta diversity**, also called "between-sample diversity", is a measurement of the distance, or difference, between samples. It involves calculating metrics such as distances or dissimilarities based on pairwise comparisons of samples so we can relate samples to each other. What stats do for us is determine the overall variation in the distance matrix and test whether groups of samples differ in community composition. 

Typically, you would generate some exploratory visualizations like ordinations and hierarchical clusterings for an overview of how your samples relate to each other. 

We’re going to use Bray-Curtis dissimilarity to cluster samples that are similar to one another based on ASV profiles. 
Bray-curtis looks at shared abundance of ASVs between two samples. 
> If samples share many taxa with similar abundances → low dissimilarity (close to 0).
> If they have very different taxa or very different abundances → high dissimilarity (close to 1)

```r
bray_dist <- phyloseq::distance(ps, method = "bray")
ord       <- ordinate(ps, method = "PCoA", distance = bray_dist)
```

### 6.1 PCoA plot

```r
p_pcoa <- plot_ordination(ps, ord, color = "Type") +
  geom_point(size = 5, alpha = 0.9) +
  geom_text(aes(label = Sample), vjust = -1, size = 3, show.legend=FALSE, position = "jitter")+
  scale_color_brewer(palette = "Set1") +
  labs(
    title    = "Beta Diversity — PCoA (Bray-Curtis)",
    subtitle = "Closer points = more similar communities",
    color    = "Type"
  ) +
  theme_bw(base_size = 12)

p_pcoa
ggsave("figures/beta_diversity_pcoa.png",p_pcoa,width = 7,height = 5,  dpi = 300)

```

> **What to expect:**
> - Plant replicates cluster tightly (Bray-Curtis ~0.18) — consistent biofilm community
> - Water replicates are moderately similar (~0.39)
> - Sediment replicates are more spread apart (~0.47), driven by Sediment_2's unusually high diversity
> - Rock replicates are completely dissimilar (BC = 1.0) — Rock_1 and Rock_2 share zero species, almost certainly a low-read-depth artefact in Rock_1


~Fin~

