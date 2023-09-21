---
layout: post
title:  "#33 Analyzing data release 2 variants with gene annotations"
date:   2023-09-21 06:00:00 -0700
author: Paul Billing-Ross
categories: jekyll update
---
## Introduction 
After uploading the variant information to BigQuery for Data Release 2 of the Million Veteran Program whole genome sequencing initiative, I wanted 

As part of Data Release 2 of the Million Veteran program whole genome sequencing initiative, Joe created a variant table from Hail, with metadata describing all of the variants we observed. In my last post, I uploaded this table to BigQuery and generated some basic summary statistics. Here, I'll describe how I've continued that effort by uploading gene annotations from Ensembl and doing some preliminary comparative analyses between our dataset and others like [1000 Genomes](https://www.internationalgenome.org/). 

## Comparing to 1000 Genomes
### Plotting minor allele frequency distribution
Looking at whole genome sequencing publications from other consortiums, I found a lot of sophisticated and complex data analysis that would seemingly require a significant amount of effort to replicate. We will maybe get to those at some point, but I certainly didn't want to start there. However, the alternate allele frequency [histogram (Figure 7a)](https://www.nature.com/articles/nature15393/figures/7) is something that only required one metrics (allele frequency) and provides a measure of validation of the "shape" of our data. Here, I've tried to replicate it with our data.

#### 1000 Genomes alternate allele frequency distribution
![1000 Genomes alternate allele frequency distribution](/assets/2023-09-21/1000genomes_alt_allele_freq_dist.png)

#### MVP Data Release 2 alternate allele frequency distribution
![MVP alternate allele frequency distribution](/assets/2023-09-21/binned_maf_distribution_histogram.png)

### Comparing SNV/Indel counts across frequency categories
In the 1000 Genomes whole genome sequencing [publication](https://pubmed.ncbi.nlm.nih.gov/36055201/) from 2022, they examine alternate allele counts by type (SNV/INDEL) and frequency bins (singleton, rare, common). I have tried to replicate a similar analysis of our data looking at variant counts (unique locus and alternate allele combinations) as well as alternate allele counts (total number of alternate alleles observed across all variants).

#### Variant counts by type and frequency category
Here is the plot of variant counts by type and frequency category for MVP Data Release 2. In addition to seeing the breakdown this also provided a validation of my SQL command, because I could check that the total number of variants matched [Joe's reported total](https://docs.google.com/presentation/d/1_Juq0AJYsRBIRHocOnLeeTBSx6ejAOHqC1hj4d1UBZY/edit#slide=id.g245fd3f6fbc_0_1606).

![Bar chart of variant counts stratified by frequency category and variant type](/assets/2023-09-21/variant_counts_snv_indel_bar.png)

#### Alternate allele counts by type and frequency category
This is a similar figure where variant counts have been replaced by alternate allele counts.

![Bar chart of alternate allele counts stratified by frequency category and variant type](/assets/2023-09-21/alternate_allele_counts_snv_indel_bar.png)

And here is figure 1B from the 1000 Genomes paper (2022). I initially thought that they were examining umber of variants (unique locus and alternate allele combinations), but reading the figure description they indicate they were looking at "cohort-level alternate allele counts", so that is what I am comparing against our data here.

![1000 Genomes allele count figure](/assets/2023-09-21/1000genomes-ac-type-frequency.jpg)

## Comparing to UK Biobank
### Analyzing transition/transversion counts

#### UK Biobank C/T conversion proportions
In the UK Biobank study they looked specifically at C/T conversions and CG (CpG) -> TG (TpG) converstions because they apparently saw a lot more of these than other possible mutations.

![Bar chart of UK Biobank C/T conversions](/assets/2023-09-21/uk-biobank-ts-tvs-bar.png)

#### MVP transitions/transversion counts
I performed a similar analysis, but looked at all possible SNV conversions and did not look at dinucleotide pairs such as mutations within CG pairs. I'm also not sure how to do this since I would need to look across rows where the position value is adjacent. But I imagine it's doable.

![Bar chart of variant counts by allele combination](/assets/2023-09-21/variant_counts_by_allele_combination.png)


## Uploading Ensembl gene annotations to BigQuery

## Counting variants in different gene functional regions

## Discuss
Join the discussion of this post on [GitHub](https://github.com/orgs/va-big-data-genomics/discussions/31)!
