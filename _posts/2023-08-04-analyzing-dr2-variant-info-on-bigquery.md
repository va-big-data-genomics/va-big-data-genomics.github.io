---
layout: post
title:  "#28 Analyzing data release 2 variant info on BigQuery"
date:   2023-08-04 06:00:00 -0700
author: Paul Billing-Ross
categories: jekyll update
---

# Hail variant information
As we approach releasing our second set of whole genome sequencing variant data, researchers will be interested in understanding which variants are included in our dataset. The rows table of our Hail matrix table includes metadata and summary statistics for each variant, generated using the Hail `variant_qc()` method. While the entire matrix table is almost 200 terabytes, the rows table is only a fraction of that size at 200 gigabytes. Sharing this information with users or potential users can allow them to better understand the dataset before trying to apply it to their research question.

# BigQuery
One option for sharing variant information is BigQuery, a high performance columnar database available on Google Cloud Platform.

## Advantages
- **Fast queries**. BigQuery scales compute resources to match the size of our data and run queries quickly.
- **Serverless**. The infrastructure for running BigQuery is maintained by Google so you don't have to set up, maintain, or configure any compute resources. And, it's always available.
- **Shareable**. BigQuery allows you to share tables publicly as well as control which columns within a table are made public. In addition to sharing your own datasets, you can also your tables with existing public tables with datasets including [gnomAD](console.cloud.google.com/bigquery?ws=!1m4!1m3!3m2!1sbigquery-public-data!2sgnomAD), (dbSNP)[console.cloud.google.com/bigquery?ws=!1m5!1m4!4m3!1sbigquery-public-data!2shuman_variant_annotation!3sncbi_dbsnp_hg38_20180418], and (ClinVar)[console.cloud.google.com/bigquery?ws=!1m5!1m4!4m3!1sbigquery-public-data!2shuman_variant_annotation!3sncbi_clinvar_hg38_20180701].
- **Machine learning**. The new BigQuery ML library promises to run machine learning algorithms through SQL queries, though I've never used.

## Obstacles
- **Cost**. Every query costs money and users need to be associated with a Google Cloud billing account to get started.
- **Access**. Users access BigQuery through the Google Cloud Platform web portal an must have a Google account associated with a billing account, to get started. For Million Veteran Program researchers, this could be a challenge.
- **SQL**. BigQuery uses the standard SQL query language and seems to have some BigQuery specific language quirks.

For our data, I think the ability to share variant data publicly is particularly valuable. It would offer potential researchers, collaborators, or researchers not yet part of the Million Veteran Program a relatively low cost (in terms of money and barrier to entry) option to explore the dataset. It's also a solution already used by another large-scale genomic sequencing program (gnomAD).

# Importing variant data into BigQuery

# Analyzing variant data in BigQuery
With the Data Release 2 variant data in BigQuery, I ran some SQL queries to explore the data and also try replicating results Joe had generated with Hail.

## How many variants are in the dataset?

```
SELECT COUNT(1) AS variant_count
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info` 
```

### Result

| Row      | variant_count |
| ----------- | ----------- |
| 1      | 663351127      |

663,351,127 variants.

## How many variants are singletons?

```
SELECT COUNT(1) AS singleton_count
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info` 
WHERE variant_qc_AC[1] = 1
```

### Result

| Row      | singleton_count |
| ----------- | ----------- |
| 1      | 318687479       |

318,687,479 of 663,351,127 (48%) variants are singletons.

## For how many variants are there no alternate alleles in the dataset?

```
SELECT COUNT(1) AS no_alt_alleles
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info`
WHERE variant_qc_AC[1] = 0
LIMIT 1000
```

### Result

| Row      | no_alt_alleles |
| ----------- | ----------- |
| 1      | 8436397     |

There are 8,436,397 variants without alternate alleles present.

## How many rare variants are present?

```
SELECT COUNT(1) AS rare_variants
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info`
WHERE variant_qc_AF[1] < 0.001
AND variant_qc_AF[1] > 0
```

### Result

| Row      | rare_variants |
| ----------- | ----------- |
| 1      | 627219244    |

627,219,244 of 663,351,127 (94.5%) variants are rare (MVP allele frequency <0.1%).

## How many variants are uncommon?

```
SELECT COUNT(1) AS uncommon_variants
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info` 
WHERE variant_qc_AF[1] < .01
AND variant_qc_AF[1] > 0.001
```

### Result

| Row      | uncommon_variants |
| ----------- | ----------- |
| 1      | 14516901    |

14,516,901 of 663,351,127 (2%) variants are uncommon (MVP allele frequency: .01% < x < 1%).

# How many variant alternate alleles are common?

```
SELECT COUNT(1) AS common_variants
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info` 
WHERE variant_qc_AF[1] > .01
```

## Result

| Row      | common_variants |
| ----------- | ----------- |
| 1      | 13178510   |

13,178,510 of 663,351,127 (1.9%) variant alleles are common.

## Get variant counts by chromosome

```
SELECT contig, COUNT(1) AS variant_count
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info` 
GROUP BY contig
```

### Results

| Row | contig | variant_count |
| -- | -- | -- |
| 1 | chr19 | 13562486 |  
| 2 | chr17 | 17934232 |  
| 3 | chr14 | 20338856 |  
| 4 | chr11 | 30797767 |  
| 5 | chr10 | 30079604 |  
| 6 | chr2 | 55235295 |  
| 7 | chr1 | 50354078
| 8 | chr20 | 14268920 |  
| 9 | chr4 | 44120441 |  
| 10 | chr7 | 35944807 |  
| 11 | chr5 | 40802899 |  
| 12 | chr3 | 45357155 |  
| 13 | chr18 | 17247139
| 14 | chr6 | 38513297 |  
| 15 | chr22 | 8765950 |  
| 16 | chrX | 33199253 |  
| 17 | chr8 | 35346863 |  
| 18 | chrY | 3602940 |  
| 19 | chr12 | 29719936 |  
| 20 | chr15 | 18658893 |  
| 21 | chr16 | 20654966 |  
| 22 | chr21 | 8443887 |  
| 23 | chr9 | 28315503 |  
| 24 | chr13 | 22085960 |

These results match [Joe's calculations](https://docs.google.com/presentation/d/1_Juq0AJYsRBIRHocOnLeeTBSx6ejAOHqC1hj4d1UBZY/edit#slide=id.g245fd3f6fbc_0_1160) performed in Hail, but should be much easier to generate.

## Get average variant call rate by chromosome

```
SELECT contig, AVG(variant_qc_call_rate) AS avg_variant_call_rate
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info` 
GROUP BY contig
```

### Results

| Row | contig | avg_variant_call_rate |
| -- | -- | -- |
| 1 | chr10 | 0.99567887972660751 |  
| 2 | chr19 | 0.99286330931143363 |  
| 3 | chr20 | 0.99528284648452714 |  
| 4 | chr14 | 0.99533632015242224 |  
| 5 | chr12 | 0.99585806523809539 |  
| 6 | chr15 | 0.99457693806272363 |  
| 7 | chr11 | 0.99599855061829679 |  
| 8 | chr1 | 0.99510411608390437 |  
| 9 | chr16 | 0.994414110412962 |  
| 10 | chr17 | 0.99427322989186062 |  
| 11 | chr22 | 0.992654612574794 |  
| 12 | chr13 | 0.99596520686626244 |
| 13 | chr6 | 0.9961985031042655 |  
| 14 | chr4 | 0.99621042682936789 |  
| 15 | chr7 | 0.99549647752316439 |  
| 16 | chr3 | 0.99633056191751879 |  
| 17 | chr21 | 0.99372139594596853 |  
| 18 | chr8 | 0.99598523462520538 |  
| 19 | chr5 | 0.99630591361731413 |  
| 20 | chrY | 0.8645258836283678 |  
| 21 | chr2 | 0.995999184525582 |  
| 22 | chr9 | 0.99443882679993179 |  
| 23 | chr18 | 0.9963189734546698 |  
| 24 | chrX | 0.954729023651225 |

## Count variants where alternate allele is only present in single individual

```
SELECT COUNT(1) AS alt_allele_in_single_person
FROM `{mvp-project}.wgs_data_release_2.hail_variant_info` 
WHERE variant_qc_homozygote_count[1] = 1 
OR variant_qc_AC[1] = 1
```

### Result

| Row      | alt_allele_present_in_single_person |
| ----------- | ----------- |
| 1      | 335112487   |

335,112,487 of 663,351,127 (50%) variants are only present in a single individual.

# Future directions
  
