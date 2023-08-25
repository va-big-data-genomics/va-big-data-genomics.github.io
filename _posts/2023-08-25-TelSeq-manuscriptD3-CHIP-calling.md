---
layout: post
title:  "#30 Updates on Telomere Length Estimation Manuscript and CHIP Calling on MVP WGS "
date:   2023-08-25 10:00:00 -0800
author: Prathima Vembu 
categories: jekyll update
---

# TelSeq Manuscript - Draft 3 

* Completed the third draft of the [TelSeq manuscript](https://docs.google.com/document/d/1jI4xpw35Q-0vUZYVQ0o_wL0b0BYvG7G_/edit#bookmark=id.30j0zll).  

# CHIP calling on MVP WGS 

## Overview

Clonal hematopoiesis of uncertain potential (CHIP) is a prevalent type of somatic mosaicism linked to notable morbidity and mortality in relation to aging. CHIP mutations may be identified from peripheral blood samples through whole-genomes, whole-exome or targeted sequencing techniques. In this [study](https://pubmed.ncbi.nlm.nih.gov/36652671/) a step wise mechanism to call CHIP calls in ~550,000 individuals from the UK Biobank complete whole-exome cohort and the All of Us Research Program initial whole genome release cohort, has been developed. 

CHIP driver genes are specific genes that have undergone changes or mutations in certain blood cells, causing those cells to behave abnormally. These genes act as "drivers" because their mutations lead to the growth and reproduction of these abnormal cells. The study uses [Mutect2](https://gatk.broadinstitute.org/hc/en-us/articles/360037593851-Mutect2) to identify putative somatic variants in 73 of the 74 canonical CHIP driver genes from the UK Biobank and All of Us CRAM files.

Mutect2 is a part of the Broad Institute GATK suite of tools that calls somatic nucleotide (SNVs) and insertions and deletions (INDELS) via local assembly of haplotypes. The pipeline was benchmarked on Google Cloud Platform (GCP) by Cuiping Pan and the initial CHIP calling was implemented on Stanford Abdominal Aortic Aneurysm (AAA) genomes and MVP Genomes (processed by Bina Technologies) using a list of CHIP sites of interest and a list of Panel of Normal SNPs. 


## Plan of action

### Phase 1

**Step 1: Establishing project infrastructure**

* Create a dedicated [GitHub Repository](https://github.com/va-big-data-genomics/chip-calling-mvp-wgs/tree/main) for CHIP calling on MVP WGS.
* Establish a project page to track and exhibit the project's progression. 


**Step 2: Initialization and execution**

* Identify and collect all the required reference files 
* Develop a structured dsub script for Mutect2
    * Identify parameters/reference files that may need to be updated
* Run Mutect2 with [Broad Institute docker image](https://hub.docker.com/r/broadinstitute/gatk/) on the CRAMs (10) and output the results to Google Cloud Storage
    * Create separate GCS storage folder for results under collaborate bucket 
* Confirm results with Alex and Cuiping 
* Other things that need to be monitored:
    * Identify the machine type used (Perform comparison of machine types?) - Identify cheapest option 
    * Estimate cost for running Mutect2 on CRAMs (10) - Use [pricing calculator](https://cloud.google.com/products/calculator) to estimate cost per sample 
    * Estimate runtime per sample 

**Step 3: Establish QC metrics**

* Identify and establish QC parameters 
* Build computational QC pipeline 

**Step 4: Generate computational insights**

* Generate computational notebook
    * Export the data into Jupyter notebook/BigQuery tables 
    * Establish methods with graphical representation of results 

**Step 5: Documentation** 

* Update GitHub repository with results, analysis notebook and methods document 

### Phase 2

* Dockerize Mutect2 
* Rerun with Phase1 CRAM (10) to confirm execution and results
* Scale up the pipeline for 1000 samples 
* Complete Step 3, 4 and 5 as in Phase 1 

### Phase 3

* Run pipeline in parallel for ~78k samples (Should we include the remaining ~30k which is a part of Data Release 2?)
* Complete Step 3, 4 and 5 as in Phase 1 


## Required files

1. --input : sample BAM/CRAM files
2. --red-index : BAI/CRAI files 
3. --reference : Reference genome (hg38.fa)
4. --normal-sample : Sample name of normal
5. --tumor-sample : Sample name of tumor
6. --output :  Ouput file in VCF format
7. --intervals - List of CHIP sites of interest
8. --panel-of-normals - VCF file of sites observed in normal
9. --germline-resource -  Population VCF of germline sequencing containing allele fractions 


## Next steps

1. Gather all reference files 
2. Create a working script for Mutect2 with updated parameters and files - Confirm Mutect2 script with Alex and Cuiping 
3. Run Mutect2 on GCP with the GATK Docker image 

