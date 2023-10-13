---
layout: post
title:  "#36 Comparing MVP release 2 to gnomAD v3 variants"
date:   2023-10-13 10:05:00 -0800
author: Paul Billing-Ross
categories: jekyll update
---

## Overview
With release 2 of Million Veteran Program whole genome sequencing data, we've identified over 660 million unique variants, or locus and alternate allele combinations. We know that many of these variants are singletons or rare variants. One question we have is: how many of these variants have been observed in other large datasets? To address that, I compared the variants we observed in the Million Veteran Program release 2 with those observed in gnomAD v3, which includes a similar number of variants at 707 million.

To investigate this question, I used our variant metadata table from Hail as well as the gnomAD v3 public tables which are all available on BigQuery.

## Shared variants between MVP and gnomAD
In order to determine how many variants are shared between MVP and gnomAD, I wrote a SQL query that considers variants to be shared between datasets if they have the same chromosome, position, reference, and alternate bases.Because the gnomAD data is split into separate tables, by chromosome, I used Python code to run this query across all chromosomes and then combine the results.

```
SELECT COUNT(*) AS shared_variants
FROM `{project_id}.wgs_data_release_2_us.hail_variant_info` AS hail
INNER JOIN `bigquery-public-data.gnomAD.v3_genomes__chr{chromosome}` AS gnomad
    ON hail.contig = 'chr{chromosome}'
    AND hail.position = gnomad.end_position
    AND hail.alleles[0] = gnomad.reference_bases
    AND hail.alleles[1] = (SELECT alt FROM UNNEST(gnomad.alternate_bases))
```

I also wrote queries to count the total number of variants in MVP and gnomAD as well as for each chromosome. The summary of that information is here. Of the 663 million MVP variants, 325 million (49%) have also been observed in gnomAD v3.

| Chromosome | MVP variants | gnomAD variants | Shared variants |
| --- | --- | --- | --- |
| All | 663,351,127 | 707,950,943 | 325,409,090 |

In Figure 1, you can also see the breakdown by chromosome. Across the autosomes there is a consistent pattern of gnomAD including slightly more variants than MVP. This is somewhat surprising given that gnomAD included 71,402 samples while we have 104,923. My guess is that this is due to us using for quality thresholds for including variants. The pattern is slightly reversed for the sex chromosomes where we see more variants than gnomAD.

![Scatterplot of shared variants by chromosomes](/assets/2023-10-13/variant_counts_mvp_gnomad.png)
Figure 1. Scatterplot of shared variant counts between MVP release 2 and gnomAD v3, stratified by chromosome.

## Comparing allele counts between MVP and gnomAD
Given that we see about 50% of the MVP variants in gnomAD, another question I had was whether those alternate alleles occurred at similar rates between MVP and gnomAD. To investigate this, I compared allele counts of shared variants between datasets. I did this by creating a temporary table to stored alternate allele counts for shared variants.

```
CREATE OR REPLACE TABLE `{project_id}.wgs_data_release_2_us.temp_mvp_gnomad_allele_counts` AS
SELECT
  hail.contig AS chromosome,
  hail.position AS start_site,
  hail.alleles[1] AS alternate_allele,
  hail.variant_qc_AC[1] AS mvp_alt_allele_count,
  (SELECT AC FROM UNNEST(gnomad.alternate_bases)) AS gnomad_allele_count
FROM `{project_id}.wgs_data_release_2_us.hail_variant_info` AS hail
JOIN `bigquery-public-data.gnomAD.v3_genomes__chr1` AS gnomad
ON hail.contig = gnomad.reference_name
  AND hail.position = gnomad.end_position
  AND hail.alleles[1] = (SELECT alt FROM UNNEST(gnomad.alternate_bases))
```

I then subsamples 10,000 variants from chromosome 1 to generate a scatterplot describing alternate allele counts between datasets (Figure 2).

![Scatterplot of allele counts](/assets/2023-10-13/alt_allele_counts_mvp_gnomad.png)
Figure 2. Scatterplot of alternate allele counts of variants shared between MVP and gnomAD datasets.

I also created a scatterplot on a log scale so that you can get a better view of variants that have low alternate allele counts.

![Scatterplot of allele counts(log)](/assets/2023-10-13/alt_allele_counts_mvp_gnomad_log.png)
Figure 3. Scatterplot of alternate allele counts of variants shared between MVP and gnomAD datasets.

Overall, I would say there is high concordance of allele counts between the datasets.

## Discuss
Join the discussion of this post on [GitHub](https://github.com/orgs/va-big-data-genomics/discussions/37)!
