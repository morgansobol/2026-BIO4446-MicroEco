# 16S Taxonomic Profiling with EMU
### A tutorial for Nanopore full-length 16S amplicon data

---
#### Emu is a relative abundance estimator for 16S genomic sequences. The method is optimized for error-prone full-length reads, but can also be utilized for short-read data.

Link to paper here: https://doi.org/10.1038/s41592-022-01520-4


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

We are creating a variable called `sample` to store the name of each file, without the folder path or the .fastq ending. For example, if `$f` is data/Plant_1.fastq, then:

- `basename` strips the folder path → Plant_1.fastq
- `.fastq` tells it to also remove the extension → Plant_1
So `sample` becomes Plant_1

We then use `$sample` to name the output folder: results/Plant_1, results/Plant_2, etc. — keeping each sample's results neatly separated.

Then we run emu on each sample, provide the path to the database, and set threads=6 (i.e. to use 6 computer nodes). 

We will run the `for` loop below from a `bash` script, let's make that. 

A `bash` script is a file containing commands that you run in the terminal. They automate tasks and make your work more efficient. We give the file a `.sh` extension because we intend to exectute the scripts in ba_sh_.

Creat the file:
```bash
nano emu.sh
```
Copy and paste everything below into file:
```bash
#!/bin/bash

for f in data/*.fastq; do
  # Extract sample name (e.g. Plant_1)
  sample=$(basename "$f" .fastq)

  emu abundance "$f" --db db/ --threads 6
done
```
A shebang (#!) is the first line in a script, starting with #!, that tells the operating system which interpreter (e.g., bash) to use to execute the file. 

Now, this is going to take ~15-25 min. In the meantime, take a break or work on the Terminus game
https://github.com/morgansobol/2026-BIO4446-MicroEco/blob/main/Tutorials/Getting-Started.md#-exercise-4-terminus-game 

### Combine all samples into one table

```bash
emu combine-outputs results/ species
```
---
Congrats!! You now have your species table. Let's take a look at the file outside of the terminal. 

