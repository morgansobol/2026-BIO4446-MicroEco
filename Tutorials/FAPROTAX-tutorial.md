# 16S Functional Profiling with FAPROTAX

---
#### FAPROTAX is a database that maps prokaryotic clades (e.g. genera or species) to established metabolic or other ecologically relevant functions, using the current literature on cultured strains.
<img width="895" height="597" alt="image" src="https://github.com/user-attachments/assets/b4068769-89b6-477b-915f-2ad06c91bc01" />



Link to website: http://www.loucalab.com/archive/FAPROTAX/lib/php/index.php?section=Home

Link to paper: https://www.science.org/doi/10.1126/science.aaf4507 


> [!IMPORTANT]
> 16S rRNA profiling _**does not**_ provide direct information about the functional capabilities or the functional genes and pathways that bacteria possess. Microbial species can have different functional capabilities, even if they are taxonomically similar. For example, different strains or species of bacteria may have different metabolic pathways. Caution is necessary when interpreting the results since they are limited by species/strain resolution and our databases. Any inferred functions are **"PUTATIVE"** or **"HYPOTHETICAL"**.


## Getting started

Navigate to the class directory `2026-MicroEco16S/` 
```bash
cd Desktop/2026-MicroEco16S/
```

Check you are in the right place with `pwd`. 

Download the FAPROTAX package.
```bash
curl -o FAPROTAX_1.2.12.zip https://pages.uoregon.edu/slouca/LoucaLab/archive/FAPROTAX/SECTION_Download/MODULE_Downloads/CLASS_Latest%20release/UNIT_FAPROTAX_1.2.12/FAPROTAX_1.2.12.zip
```

Unzip the folder.
```bash
unzip FAPROTAX_1.2.12.zip
```

Move into this new directory.
```bash
cd FAPROTAX_1.2.12
```

Download the following python script into the `FAPROTAX_1.2.12`directory
https://github.com/morgansobol/2026-BIO4446-MicroEco/blob/main/Tutorials/emu-to-faprotax-fix.py

Activate the emu conda environment
```bash
conda activate emu
```

## Analyzing the data

We first need to convert the EMU output to a combatible output for FAPROTAX using this script.
```bash
python emu-to-faprotax-fix.py ../results/emu-combined-species.tsv faprotax_input.tsv 
```

Now we can run FAPROTAX.
```bash
python collapse_table.py -i faprotax_input.tsv -o FAPROTAX-output-by-site.tsv -g FAPROTAX.txt -d "taxonomy" -c "#" -b -v -r FAPROTAX-output-summary.tsv
```


