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

## Plotting minor allele frequency distribution
![Histogram of minor allele frequency distribution](/assets/2023-09-21/binned_maf_distribution_histogram.png)

## Analyzing transition/transversion counts
![Bar chart of variant counts by allele combination](/assets/2023-09-21/variant_counts_by_allele_combination.png)

## Comparing SNV/Indel counts across frequency categories
![Bar chart of variant counts stratified by frequency category and variant type](/assets/2023-09-21/variant_counts_snv_indel_bar.png)
![Bar chart of alternate allele counts stratified by frequency category and variant type](/assets/2023-09-21/alternate_allele_counts_snv_indel_bar.png)

## Uploading Ensembl gene annotations to BigQuery

## Counting variants in different gene functional regions

## Discuss
Join the discussion of this post on [GitHub](https://github.com/orgs/va-big-data-genomics/discussions/31)!
