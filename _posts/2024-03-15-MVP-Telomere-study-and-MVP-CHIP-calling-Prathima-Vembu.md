---
layout: post
title:  "#42 Results from LTL estimation on 103,763 WGS genomes and CHIP calling on 100 genomes"
date:   2024-03-15 10:05:00 -0800
author: Prathima Vembu 
categories: jekyll update
---

# Telomere length estimation in MVP WGS data 
Leukocyte telomere length estimation for 103,763 MVP WGS samples has been completed. The analysis notebook will soon be uploaded to the mvp-telomere-analysis git repo.  Regarding the number of samples:

- 8562 samples from LTL Pilot study are not a part of WGS DR2. 
- 808 samples that were a part of WGS DR2 were not a part of the LTL study. 

## Sample count:
|||
|-|-|
|Number of samples in WGS DR2|104,923| 
|Number of samples that have demographic information|104,901|
|Number of samples in LTL Pilot study|84,677|
|Number of samples in LTL 2nd run|31,873|
|Number of missing ID's between WGS DR2 and LTL|808|
|Number of samples in TelSeq 3rd run|788 (808-20)|

*Note: 20 samples from WGS DR2 do not have cram locations available.*

## 1. Filteration:
- In total, Telomere length estimation haas been completed for 117,338 samples. 
- After removing the 8562 genomes (samples in Telseq Pilot but NOT a part of WGS DR2), the number of samples drops down to 108,776. 
- From the 108,776 samples, the number of samples common between WGS DR2 and TelSeq = 104,903. <br>

Hare distribution within the 104,903 samples:

|hare|count| 
|--|--|
|EUR|72,916|
|AFR|24,612|
|ASN|5,548|
|HIS|687|
|NA|1118|

- NA (1118) samples have been removed for all subsequent studies, dropping the number of samples to **103,763**.
- 22 samples do not have any demographic information.


## 2. Distribution of 'Age' 

#### 2.1. Across 103,763 MVP WGS samples
![Distribution of 'Age' across 103,763 MVP WGS samples](/assets/2024-03-15/Distribution-of-Age-all.png)

||Pilot study|Final study|
|-|-|-|
|Mean age|65 years|65 years|
|Standard deviation|14 years |13 years |
|Minimum age|20 years|14 years|
|Maximum age|101 years|99 years|


## 3. Distribution of estimated LTL across 103,763 MVP WGS samples

![Distribution of estimated LTL across 103,763 MVP WGS samples](/assets/2024-03-15/Distribution-of-estimated-LTL.png.png)

Mean LTL for the Pilot study was 2.02kb. 

## 4. Correalation between 'Age' and 'LTL'

#### 4.1. Across all 103,763 sampples 
![Correalation between 'Age' and LTL across all samples](/assets/2024-03-15/Correlation_age_ltl_all.png)

Correaltion co-efficient between Age and LTL in the Pilot study = -0.26 (p value < 1e-16)

#### 4.2. Stratified by Sex
![Correalation between 'Age' and LTL - stratified by Sex](/assets/2024-03-15/Correlation_age_ltl_sex.png)

Correaltion co-efficient between Age and LTL in the Pilot study for each sex:

- Male = -0.24 (p value < 1e-16)
- Female = -0.24 (p value = 4.8252e-68)

#### 4.3. Stratified by HARE
![Correalation between 'Age' and LTL - stratified by HARE](/assets/2024-03-15/Correlation_age_ltl_HARE.png)

Correaltion co-efficient between Age and LTL in the Pilot study for each HARE group:

- AFR = -0.23  (p value = 1.36e-225)
- EUR = -0.27  (p value <1e-16)
- HIS = -0.22   (p value <1e-16)
- ASN = -0.21  (p value <1e-16)

## 5. Comparing estimated LTL across 103,763 samples 

#### 5.1. Stratified by Sex
![Comparing estimated LTL - stratified by Sex](/assets/2024-03-15/Comparing_LTL_by_Sex.png)

Mean estimated LTL in Pilot study, by sex:

- Male = 2.01kb 
- Female = 2.16kb 

#### 5.2. Stratified by HARE
![Comparing estimated LTL - stratified by HARE](/assets/2024-03-15/Comparing_LTL_by_HARE.png)

Mean estimated LTL in Pilot study, per HARE group:

- AFR = 2.07kb 
- EUR = 2.01kb
- HIS = 2.03kb
- ASN = 2.0kb

## 6. Variation in correlation between Age and LTL varies every 20 years 

![Understanding how correlation between Age and LTL varies, every 20 years](/assets/2024-03-15/20-year-correlation-age-ltl.png)

Next steps:

- Push completed notebook and figures to the GitHub repository
- Share telomere results with DACS team 

# CHIP calling in MVP WGS data 

GATK Mutect2 pipeline has been successfully completed for 100 samples. The pipeline was executed using the docker image [broadinstitute/gatk:latest](https://hub.docker.com/r/broadinstitute/gatk). The docker image is bulky as it contains all the tools that are a part of the GATK suite, and hence requires a 'boot-disk-space' allocation of 20Gb. Along with this, the tool also requires a reference dictionary file used to describe a reference genome assembly. The other depencies include the input CRAM, index  CRAI file, reference fasta file, list of CHIP driver genes and the germline resource file which is a population VCF of germline sequencing containing allele fractions. 

### Mutect2 script
```
#!/bin/bash 

#Extract file name
export CRAM_NAME="$(basename $CRAM .cram).vcf"

#Run Mutect2
gatk Mutect2 \
    -I $CRAM \ 
    -L $CHIP_GENES \ 
    -R $REF \ 
    --germline-resource $GERM_RES \
    --genotype-germline-sites \ 
    -O "$(dirname "${OUTPUT}")/${CRAM_NAME}"

```
### dsub job script
```
dsub \
--provider google-v2 \
--project PROJECT_ID \
--regions us-west1 \
--machine-type n1-standard-1 \
--logging gs://PATH/TO/LOG/ \
--name chip_mvp \
--image broadinstitute/gatk:latest \
--boot-disk-size 20 \
--task "gs://PATH/TO/task.tsv" \
--script "gs://PATH/TO/mutect2_script.sh"
```
### Tab-seperated task file 
The TSV file utilized in the batch dsub script serves as a structured data input file, containing tab-separated values organized in rows and columns. Each row represents a distinct task or job to be executed by the dsub command, with each column corresponding to specific parameters or attributes associated with the tasks. For ease of reading the file has been transformed here. Ideally, column1 (Parameter) woud be row1 and column2 (File path) would be row2.  

|Parameter|File path|
|--|--|
|--input CRAM|gs://PATH/TO/.cram|
|--input CRAI|gs://PATH/TO/.cram.crai|
|--input CHIP_GENES|gs://PATH/TO/CHIP_NEJM2017_74genes.bed|
|--input GERM_RES|gs://gatk-best-practices/somatic-hg38/af-only-gnomad.hg38.vcf.gz|
|--input GERM_RES_INDEX|gs://gatk-best-practices/somatic-hg38/af-only-gnomad.hg38.vcf.gz.tbi|
|--input REF|gs://genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta|
|--input REF_FAI|gs://genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta.fai|
|--input REF_DICT|gs://genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.dict|
|--output OUTPUT|gs://PATH/TO//Output/*.vcf|


VCF files for the initial 100 samples have been generated and are on GCS. The next steps would be:

- Extract Mutect2 compute time for 100 samples from the log files.
- Cost estimation for 100 samples. 
- Update CHIP calling GitHub repository with scripts.
- Share results for pilot with Alex and team to discuss next steps. 

---

Join the discussion [here]!
