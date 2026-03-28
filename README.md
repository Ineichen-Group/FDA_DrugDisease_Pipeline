# FDA_DrugDisease_Pipeline

This project is part of a larger effort to track drugs from preclinical (animal) studies to clinical application. In this context, the FDA dataset is filtered to include only drugs overlapping with a separate preclinical dataset.

The pipeline itself is modular and can be adapted to other drug sets or used on the full FDA dataset with minimal changes.

## Setup

### Create environments
Create the conda environments:

```
conda env create -f torch_hf_environment.yml
```

```
conda env create -f vllm_environment.yml
```

> **Note:** The `vllm` environment is used in Step 3. It requires access to a GPU (CUDA-compatible) for running step 3.

### Download pre-computed data

Download the named entity linking resources from [https://zenodo.org/records/19287944](https://zenodo.org/records/19287944). These files contain precomputed ontology embeddings and mappings required for drug and disease normalization

After downloading, place them in the repository as follows:
```
entity_linking/data/
                ├── mondo/
                │   ├── embeddings/
                │   ├── mondo_term_id_pairs.json
                │   └── mondo_id_to_term_map.json
                └── umls/
                    ├── embeddings/
                    ├── umls_term_id_pairs_combined.json
                    └── umls_id_to_term_map.json 
```

## Step 1: Download and prepare FDA drug + label data

First, download the two openFDA datasets and save them in ./data:

- [Drugs@FDA download](https://open.fda.gov/apis/drug/drugsfda/download/)
- [Drug Label download](https://open.fda.gov/apis/drug/label/download/)

> **Note:** The Drug Label dataset is split into multiple parts. All files must be downloaded to obtain a complete dataset.

### What happens in this step

This step builds a single FDA drug table by combining application data from **Drugs@FDA** with text and metadata from **Drug Label**.

### 1. Load Drugs@FDA applications

The pipeline reads the `drug-drugsfda-0001-of-0001.json` file and keeps only records with an original submission:

- filters `submissions` for `submission_type == "ORIG"`
- extracts core application metadata such as:
  - `application_number`
  - `sponsor_name`
  - `submission_status`
  - `submission_status_date`
  - `submission_class_code_description`
  - `year`
- extracts product fields from the first listed product:
  - `brand_name`
  - `dosage_form`
  - `route`
  - `marketing_status`
  - `active_ingredients`
- flattens selected `openfda` fields when available

This produces the initial dataframe `df_orig`.

### 2. Load FDA label files

The pipeline then reads the downloaded drug label files (`drug-label-0001-of-0013.json` to `drug-label-0013-of-0013.json`).

For each label record, it:

- skips records without an `application_number`
- extracts label metadata:
  - `label_brand_name`
  - `label_generic_name`
  - `label_manufacturer_name`
  - `label_substance_name`
- extracts the full `indications_and_usage` text
- stores a short version in `indications_first_sent`
- searches `clinical_studies` text for NCT trial IDs

### 3. Merge labels into the FDA application table

The label data is merged into `df_orig` using `application_number`.

This enriches each FDA application with:

- label names and manufacturer info
- indication text
- linked clinical trial IDs when present

### 4. Save the combined dataset

The merged result is saved as:

- `FDA_drugs_labels_full.csv`


### 5. Keep only approved applications

Next, the pipeline keeps only approved original applications:

- filters `submission_status == "AP"`

This creates the approval-only dataset used in later steps.

### 6. Build normalized drug term strings

To prepare for ontology mapping, the code combines drug naming fields into one text field:

- `active_ingredients`
- `label_substance_name`
- `brand_name`

This combined field is stored as:

- `active_ingredients_substance_brand`

Then it splits combination drugs into separate terms and explodes them into one row per ingredient term:

- `active_ingredients_split`

This makes downstream normalization easier.

### 7. Map ingredient terms to UMLS

The exploded ingredient terms are passed to the entity-linking step.

This maps each drug term to a normalized UMLS concept and adds:

- `drug_umls_term_norm`
- `drug_umls_termid`

The mapped output is saved as:

- `FDA_full_AP_mapped_active_ingredients.csv`

### 8. Add parent UMLS concepts

Finally, the pipeline optionally expands mapped drug concepts to parent concepts using precomputed UMLS parent mappings.

For rows with a valid parent:

- parent UMLS IDs and labels are added
- duplicate parent-expanded rows are appended back into the dataset

This creates the final output:

- `FDA_full_AP_mapped_active_ingredients_with_parent.csv`

### Output of this step

At the end of this step, you have a processed FDA drug table that includes:

- original FDA application metadata
- linked label text and indication text
- extracted trial IDs
- one row per normalized drug term
- UMLS concept mappings
- optional parent concept expansion

## Step 2: Filter FDA drugs to those observed in the preclinical dataset

### What happens in this step

The goal of this step is to keep only FDA drug records that also appear in the preclinical dataset, and then carry forward their FDA indication text for downstream processing. 

### 1. Load the processed FDA file

The pipeline loads the FDA table created in the previous step:

- `FDA_full_AP_mapped_active_ingredients_with_parent.csv`

This file contains approved FDA drugs, mapped UMLS drug concepts, and label-derived indication text.

### 2. Load preclinical drug terms

Next, the pipeline loads the normalized drug terms from the preclinical dataset:

- `unique_drug_terms_292514.csv`

These terms represent the drug concepts already observed in the preclinical pipeline.

### 3. Match FDA drugs to preclinical drugs

The code matches the two datasets using normalized UMLS drug labels:

- preclinical: `merged_umls_label`
- FDA: `drug_umls_term_norm`

To make matching more robust, both sides are normalized to:

- uppercase
- stripped whitespace

This merge is used to identify **which FDA drugs are also present in the preclinical dataset**.

### 4. Keep only FDA records with a preclinical match

After merging, the pipeline keeps only rows with a valid FDA `application_number`.

This effectively filters the FDA dataset down to:

- FDA-approved drugs that match a drug concept seen in the preclinical dataset

So yes — **this step is specifically filtering FDA to only the subset relevant to the preclinical drug space**.

### 5. Carry forward FDA indication text

For the matched FDA drugs, the pipeline keeps FDA metadata and label text, especially:

- `indications_first_sent`
- `indications_and_usage`
- `clinical_studies`

This is the key enrichment step:  
for drugs that overlap with the preclinical dataset, the pipeline attaches their FDA indication text.

### 6. Keep only rows with indication text

The pipeline then filters again to keep only matched FDA records where indication text is available.

These rows are used in the next stage for indication parsing / LLM processing.

### Output of this step

At the end of this step, you have a dataset of:

- drugs found in the preclinical dataset
- that also have matching FDA records
- with FDA indication text attached when available

Saved outputs include:

- `df_ds_drugs_with_FDA_info.csv` — matched preclinical ↔ FDA drug records
- `drugs_with_indications.csv` — matched records that also contain indication text
- chunked versions for batch processing

## Step 3: Extract disease indications from FDA label text using an LLM

### What happens in this step

The goal of this step is to turn FDA indication text into structured disease or condition terms.

In the previous step, the pipeline kept only:

- FDA drugs that match the preclinical dataset
- rows that contain FDA indication text

This step uses an LLM to read that indication text and extract the diseases or clinically relevant conditions being treated.

### 1. Load the local LLM

The pipeline loads a locally stored instruction model with `vLLM` in offline mode. The model is `DeepSeek-R1-Distill-Qwen-32B`.

This allows the extraction to run without external API calls and uses a fixed local checkpoint for reproducibility.

### 2. Define the extraction prompt

A structured prompt is created for disease extraction from FDA `INDICATIONS AND USAGE` text.

The prompt tells the model to return only disease / condition entities, for example:

- diseases
- disorders
- syndromes
- clinically relevant symptoms when they are explicit treatment targets

The response must be returned as JSON inside `<json>...</json>` tags.

### 3. Parse and validate model output

The notebook includes helper functions to:

- extract JSON from the model response
- recover JSON even if the model adds extra text
- validate that the extracted disease terms are actually grounded in the original indication text

If the model output is incomplete or not supported by the text:

- the code retries
- it falls back from `indications_first_sent` to the full `indications_and_usage` text

If extraction still fails, the row gets an empty result.

### 4. Apply the LLM to FDA indication rows

The input file is:

- `drugs_with_indications.csv`

This file contains FDA-matched preclinical drugs with available indication text.

For each row, the LLM extracts disease targets from the indication text and stores them in:

- `disease_from_indications`

Examples of extracted outputs include:

- `advanced renal cell carcinoma`
- `pain`
- `acute pain|chronic pain`

### 5. Resume from prior runs

Because this step is long-running, the pipeline supports resuming from an earlier output file.

It does this by:

- loading previously processed rows using `unique_id`
- separating already completed rows from rows still to process
- running the LLM only on the remaining subset

This avoids repeating work if the job is interrupted.

### 6. Save the cleaned disease extraction output

The extracted disease labels are written back to the dataset and saved to an output file such as:

- `drugs_with_indications_LLM_cleaned_20260131.csv`

This file contains:

- the matched FDA/preclinical drug row
- the original indication text
- the LLM-extracted disease or condition labels

### Output of this step

At the end of this step, you have a dataset where:

- FDA drugs are already restricted to those overlapping with the preclinical dataset
- FDA indication text has been converted into structured disease labels
- the results are ready for downstream disease normalization or matching
## Step 4: Convert extracted FDA indications into normalized drug–disease pairs

### What happens in this step

The goal of this step is to turn the LLM output into a structured set of FDA drug–disease associations.

In the previous step, each FDA record already had:

- a matched preclinical drug
- FDA indication text
- LLM-extracted disease strings in `disease_from_indications`

This step expands those disease strings, builds one drug–disease pair per row, and normalizes the disease side to an ontology.

### 1. Load LLM-cleaned indication output

The pipeline loads:

- `drugs_with_indications_LLM_cleaned_20260131.csv`

It keeps the main fields needed for drug–disease pairing, including:

- `unique_id`
- `merged_umls_label`
- `year`
- `application_number`
- `active_ingredients_split`
- `disease_from_indications`

### 2. Expand one disease per row

The LLM output stores multiple extracted indications in a single pipe-separated field:

- `disease_from_indications`

This step splits that field on `|` and explodes it so that each row contains:

- one drug
- one disease
- one approval/application context

It also standardizes disease strings by lowercasing and trimming whitespace.

### 3. Aggregate across FDA documents

After expansion, the pipeline groups by:

- drug (`merged_umls_label`)
- disease

For each drug–disease pair, it keeps:

- `earliest_year` = earliest FDA approval year observed for that pair
- `documents` = list of FDA application numbers supporting that pair

This creates a consolidated FDA drug–disease table across all matched FDA records.

The output is saved as:

- `FDA_drug_disease_pairs_all_years.csv`

### 4. Normalize diseases to MONDO

Next, the disease column is mapped to a disease ontology using the entity-linking pipeline.

The mapping step uses:

- `mapping_type = "disease"`
- `col_to_map = "disease"`

This links the extracted disease strings to normalized MONDO terms and produces:

- `disease_mondo_term_norm`

The mapped output is saved as:

- `FDA_drug_disease_pairs_all_years_mapped_drug_disease.csv`

### 5. Combine original pairs with mapped disease terms

The pipeline then combines:

- the original FDA drug–disease pair table
- the normalized disease mapping output

This creates a dataset where each row contains both:

- the original extracted disease text
- the normalized MONDO disease label

### 6. Collapse duplicate normalized pairs

Some drug–disease pairs may appear multiple times because:

- the same indication is supported by multiple FDA applications
- several extracted disease strings map to the same normalized disease concept

To clean this up, the pipeline groups by:

- `merged_umls_label`
- `disease_mondo_term_norm`

and keeps:

- the earliest observed year
- the merged original disease strings
- the supporting FDA application documents

This produces the final non-redundant FDA drug–disease mapping table.

### Output of this step

At the end of this step, you have a normalized set of FDA drug–disease pairs that includes:

- the matched drug from the preclinical-aligned FDA subset
- extracted indication-derived diseases
- normalized MONDO disease concepts
- earliest FDA year for the pair
- supporting FDA application numbers

Saved outputs include:

- `FDA_drug_disease_pairs_all_years.csv`
- `FDA_drug_disease_pairs_all_years_mapped_drug_disease.csv`
- `FDA_drug_disease_pairs_mapped_both.csv`