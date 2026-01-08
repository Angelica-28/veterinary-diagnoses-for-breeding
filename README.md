# Veterinary-diagnoses-for-breeding
This repository contains the scripts used to analyze veterinary diagnoses derived from electronic recording systems in order to estimate the heritability of selected bovine pathologies. The pipeline includes data preprocessing, phenotype definition, and genetic parameter estimation.
# Mammary Pathology Phenotype Pipeline

This repository contains scripts to process veterinary treatment data, functional control records, and animal registry data to generate a **mammary health phenotype** per cow and lactation.

The pipeline:

1. Imports treatment data from electronic records.
2. Filters mammary-related treatments and excludes oxytocin.
3. Merges the dataset with the animal registry to keep all animals.
4. Merges functional control records.
5. Assigns the binary phenotype:
   - `PAT = 1` to the control closest to the treatment, 0 otherwise
   - `T = 1` if the cow received a dry-off treatment in that lactation, 0 otherwise
6. Builds the final dataset with columns:

- `MATR` = animal ID
- `NL` = lactation number
- `AZ` = herd/azienda code
- `PAT` = 1 if at least one mammary event, 0 otherwise
-  `T` = 1 if at least one mammary event, 0 otherwise
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
```
---
## Overview of the Workflow

The pipeline script `01_build_mammary_dataset.R` performs the following steps:

### Step 1: Load required libraries
We use `dplyr` for data manipulation, `readxl` to read Excel files, `stringr` for text cleaning, and `lubridate` for handling dates.

```r
library(dplyr)
library(readxl)
library(stringr)
library(lubridate)
```

### Step 2: Define input files
Set the paths to your input files:

`veterinary_treatments.xlsx` → veterinary treatment records

`animal_registry.xlsx` → animal registry

`functional_records.xlsx` → functional control data

```r
treatment_file <- "veterinary_treatments.xlsx"
registry_file   <- "animal_registry.xlsx"
functional_file <- "functional_records.xlsx"
```

### Step 3 – Import and filter veterinary treatments

The treatment dataset is filtered to retain only:

- `Mammary pathologies`

- `Dry-off treatments`

Treatments containing oxytocin are excluded.

```r
treatments <- read_excel(treatment_file) %>%
  mutate(
    TIPO_DIAGNOSI_clean = str_trim(str_to_lower(TIPO_DIAGNOSI)),
    PRINCIPIO_ATTIVO_clean = str_trim(str_to_lower(PRINCIPIO_ATTIVO)),
    MARCA_AURICOLARE = str_trim(MARCA_AURICOLARE)
  ) %>%
  filter(TIPO_DIAGNOSI_clean %in% c("patologie mammarie", "trattamento per asciutta")) %>%
  filter(!str_detect(PRINCIPIO_ATTIVO_clean, "ossitocina"))
```

### Step 4 – Import animal registry and merge

All animals in the registry are retained, even if no treatments are recorded.

```r
registry <- read_excel(registry_file) %>%
  mutate(MARCA_AURICOLARE = str_trim(MARCA_AURICOLARE))

data_merged <- registry %>%
  left_join(treatments, by = "MARCA_AURICOLARE")
```
### Step 5 – Import functional control data and merge

Functional records are merged by animal ID.
```r
functional <- read_excel(functional_file) %>%
  mutate(matr = str_trim(matr))

data_final <- data_merged %>%
  left_join(functional, by = c("MARCA_AURICOLARE" = "matr")) %>%
  mutate(
    DT_SOMMINISTRAZIONE = dmy(DT_SOMMINISTRAZIONE),
    data = dmy(data)
  )
```
### Step 6 – Assign pathology to the closest functional control

For each treatment, the closest functional control date is identified and assigned `PAT` = 1.

The corresponding treatment and control dates are stored.
```r
assign_pat <- function(df) {
  df$PAT <- 0
  df$closest_control_date <- as.Date(NA)
  df$closest_treatment_date <- as.Date(NA)

  tr <- which(!is.na(df$DT_SOMMINISTRAZIONE))
  if(length(tr) == 0) return(df)

  for(i in tr){
    diffs <- abs(as.numeric(df$data - df$DT_SOMMINISTRAZIONE[i]))
    j <- which.min(diffs)
    df$PAT[j] <- 1
    df$closest_control_date[j] <- df$data[j]
    df$closest_treatment_date[j] <- df$DT_SOMMINISTRAZIONE[i]
  }

  df
}

data_final <- data_final %>%
  group_by(MARCA_AURICOLARE) %>%
  group_modify(~ assign_pat(.x)) %>%
  ungroup()
```
### Step 7 – Build lactation phenotype

For each cow and lactation, the phenotype is defined as:

 - `PAT` = 1 if at least one pathology is present

 - `PAT` = 0 otherwise

Treatment and control dates are retained.
```r
final_dataset <- data_final %>%
  group_by(MARCA_AURICOLARE, NL, CODICE) %>%
  summarise(
    PAT = ifelse(sum(PAT, na.rm = TRUE) > 0, 1, 0),
    DT_SOMMINISTRAZIONE = min(closest_treatment_date, na.rm = TRUE),
    DT_CONTROLLO = min(closest_control_date, na.rm = TRUE)
  ) %>%
  ungroup() %>%
  rename(MATR = MARCA_AURICOLARE, AZ = CODICE)
```
### Step 8 – Save final dataset

The final dataset contains:

- `MATR`

- `NL`

- `AZ`

- `PAT`

- `DT_SOMMINISTRAZIONE`

- `DT_CONTROLLO`
```r
write.csv(final_dataset,
          "output/final_mammary_phenotype.csv",
          row.names = FALSE)
```
Final Output Structure
```r
MATR	NL	AZ	PAT	T	DT_SOMMINISTRAZIONE	DT_CONTROLLO
IT000	1	001	1	1	2022-08-03	2022-08-03
IT000	2	001	0	1	NA	NA
IT001	1	002	0	0	NA	NA
IT002	1	003	1	0	2022-07-15	2022-07-16
```
