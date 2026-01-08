# Veterinary-diagnoses-for-breeding
This repository contains the scripts used to analyze veterinary diagnoses derived from electronic recording systems in order to estimate the heritability of selected bovine pathologies. The pipeline includes data preprocessing, phenotype definition, and genetic parameter estimation.
# Mammary Pathology Phenotype Pipeline

This repository contains scripts to process veterinary treatment data, functional control records, and animal registry data to generate a **mammary health phenotype** per cow and lactation.

The pipeline:

1. Imports treatment data from electronic records.
2. Filters mammary-related treatments and excludes oxytocin.
3. Merges the dataset with the animal registry to keep all animals.
4. Merges functional control records.
5. Assigns the binary phenotype `PAT = 1` to the control closest to the treatment.
6. Builds the final dataset with columns:

- `MATR` = animal ID
- `NL` = lactation number
- `AZ` = herd/azienda code
- `PAT` = 1 if at least one mammary event, 0 otherwise
- `DT_SOMMINISTRAZIONE` = treatment date of the closest event
- `DT_CONTROLO` = functional control date closest to treatment

The final dataset is ready for heritability analysis or genetic evaluation.

---

## How to use

1. Place your input files in the repository folder:

- `veterinary_treatments.xlsx`
- `animal_registry.xlsx`
- `functional_records.xlsx`

2. Run the script:

```r
source("01_build_mammary_dataset.R")
