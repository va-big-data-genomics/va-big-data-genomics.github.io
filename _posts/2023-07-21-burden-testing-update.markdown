---
layout: post
title:  "#27 Burden testing progress update 2"
date:   2023-07-21 12:25:00 -0700
author: Daniel Cotter 
categories: jekyll update
---

### Filtering to putative loss-of-function variants

As mentioned in [my last blog post](https://va-big-data-genomics.github.io/jekyll/update/2023/06/09/burden-testing-progress-update.html), I am testing for putative loss-of-function (pLoF) variants, based on the [gnomAD variants database](https://hail.is/docs/0.2/datasets/schemas/gnomad_genome_sites.html#schema-3-1-2-grch38)'s "most_severe_consequence" field. In a comment on my last blog post, [Jenny recommended](https://github.com/orgs/va-big-data-genomics/discussions/26#discussioncomment-6136380) using the terms "start_lost", "stop_gained", "splice_acceptor_variant", and "splice_donor_variant" to match gnomAD's list of terms, so I annotated the 10k genomes matrix table with gnomAD's annotations and filtered to the variants matching that list of most severe consequences.

### Job hangs at grouping & aggregation step

When I ran the algorithm with these values, the job ran successfully until the grouping & aggregation step:

```
burden = (
    genomes
    .group_rows_by(genomes.vep_info.gene_symbol)
    .aggregate(n_variants = hl.agg.count_where(genomes.GT.n_alt_alleles() > 0))
)
```

At this point it would hang, running indefinitely without significant CPU or memory usage until I killed the job and forcibly downsized the cluster. There was no error message, since the job didn't crash, and there was nothing useful or revealing in the logs. I was stumped by this particular problem for a couple of weeks, enlisting support from the [Hail forum](https://discuss.hail.is/t/linear-regression-hanging-help-needed/3447/8) and then Paul and Joe. Joe mentioned he'd seen this behavior before when a matrix table was empty, which sounded crazy to me, because an empty table should finish *immediately*, right? But I recalled a puzzling result I got when I counted how many distinct genes were in the dataset after joining to the Human Genome Diversity Project dataset:


```
>>> genomes.aggregate_rows( hl.agg.counter( genomes.vep_info.gene_symbol ))
{None: 72723}
```

This means the entire dataset after annotating with the HGDP dataset had an NA value for the gene symbol. Paul and Joe asked why I was using the HGDP dataset, and the only reason I had was that Jina Song, who I replaced on the MVP team, had used it in the notebook on which I had based the burden testing algorithm. This is a sample of the HGDP flat file she'd used:

| variant         | gene_symbol | csq                                |
| --------------- | ----------- | ---------------------------------- |
| chr1:17379:G:A  | MIR6859-1   | mature_miRNA_variant               |
| chr1:95068:G:A  | AL627309.1  | intron_variant                     |
| chr1:111735:C:A | AL627309.1  | intron_variant                     |

The problem is that Hail was joining our variants to HGDP's variants, and none of them happened to coincide, so the HGDP side of the join was essentially null. Paul and Joe suggested using a gene, rather than variant, annotation database, which would give a start and end position on the chromosome instead of a single point and thus be much more likely to match our variants to the gene in which they were found.

I did this, using Hail's `import_gtf()` function to import Ensembl's gene annotation database and filtering down to just the gene features:

```
ensembl_ht = hl.experimental.import_gtf('gs://gbsc-gcp-project-mvp-wgs-data-release-2/burden-testing/Homo_sapiens.GRCh38.110.gtf',
                           reference_genome='GRCh38',
                           skip_invalid_contigs=True) # chrMT causes the import to fail if skip_invalid_contigs=False
ensembl_ht = ensembl_ht.key_by('interval')
ensembl_ht = ensembl_ht.filter( ensembl_ht.feature == 'gene' )
```

When I used this table to annotate our variants, the group and aggregate step finished, and the rest of the algorithm could run.

### Slimming down the genomes table
The algorithm was still taking a long time to run, so I moved all the row-filtering steps to as early in the script as I could and dropped all row, column, and entry fields that were not used to produce the final results. This minimized the table's size before the computationally intensive steps; that is, the counting of alternate alleles per gene (see "grouping & aggregation" above) and the linear regression. By filtering on allele frequency (<1%), number of alternate alleles (>0), and most severe consequence (loss of function), then dropping all the bulky gnomAD and Ensembl annotations, not to mention the many fields that were not involved in the actual computations, I was able to reduce the 10k genome matrix table to 4.24 MiB:

Python:
```
>>> genomes_mt = genomes_mt.filter_rows( hl.is_missing( genomes_mt.gene ), keep=False )
>>> genomes_mt = genomes_mt.select_rows( genomes_mt.gene )
>>> genomes_mt = genomes_mt.select_cols( genomes_mt.super_pop )
>>> genomes_mt = genomes_mt.select_entries( genomes_mt.GT )
>>> genome_mt.describe()
----------------------------------------
Global fields:

----------------------------------------
Column fields:
    's': str
    'super_pop': str
----------------------------------------
Row fields:
    'locus': locus<grch38>
    'alleles': array<str>
    'gene': array<str>
----------------------------------------
Entry fields:
    'GT': call
----------------------------------------
Column key: ['s']
Row key: [['locus', 'alleles']]
----------------------------------------
```

Shell:
```
% gsutil du -s -h gs://gbsc-gcp-project-mvp-wgs-data-release-2/burden-testing/genomes-10k-slim.mt
4.24 MiB     gs://gbsc-gcp-project-mvp-wgs-data-release-2/burden-testing/genomes-10k-slim.mt
```

### Results
I ran the algorithm again on the 10k dataset, with the following results:

| gene     |     n |    sum_x | y_transpose_x |      beta | standard_error |    t_stat |  p_value |
|----------|-------|----------|---------------|-----------|----------------|-----------|----------|
| "CPNE6"  | 10371 | 3.50e+01 |      2.50e+03 |  1.99e+00 |       4.55e-01 |  4.36e+00 | 1.29e-05 |
| "ADAP1"  | 10371 | 1.44e+02 |      9.91e+03 | -9.61e-01 |       2.24e-01 | -4.29e+00 | 1.78e-05 |
| "CYTH4"  | 10371 | 4.00e+00 |      3.02e+02 |  5.59e+00 |       1.34e+00 |  4.16e+00 | 3.17e-05 |
| "DEPDC4" | 10371 | 2.30e+01 |      1.55e+03 | -2.33e+00 |       5.61e-01 | -4.16e+00 | 3.23e-05 |
| "FBXO22" | 10371 | 1.00e+00 |      8.07e+01 |  1.08e+01 |       2.69e+00 |  4.01e+00 | 6.03e-05 |
| "CUL4A"  | 10371 | 1.30e+01 |      8.67e+02 | -2.89e+00 |       7.46e-01 | -3.87e+00 | 1.08e-04 |
| "ECHDC3" | 10371 | 3.00e+00 |      1.92e+02 | -5.94e+00 |       1.55e+00 | -3.83e+00 | 1.29e-04 |
| "ASPH"   | 10371 | 1.50e+01 |      1.00e+03 | -2.60e+00 |       6.95e-01 | -3.74e+00 | 1.82e-04 |
| "MFN1"   | 10371 | 2.00e+00 |      1.26e+02 | -7.11e+00 |       1.90e+00 | -3.74e+00 | 1.86e-04 |
| "GPR78"  | 10371 | 4.30e+01 |      2.93e+03 | -1.52e+00 |       4.11e-01 | -3.71e+00 | 2.09e-04 |

These do not match Genebass's "[pLoF gene burden associations with height male custom](https://app.genebass.org/gene/undefined/phenotype/continuous-height_male_custom-males--custom?resultIndex=gene-manhattan&resultLayout=full)". They also do not match Bryan's top ten:

| ID                          | LOG10P     |
| --------------------------- | ---------- |
| NOC2L.Mask1.0.01            | 2.79E-07   |
| HSPA9.Mask1.0.01            | 4.39E-05   |
| SIRPD.Mask1.singleton       | 6.55E-05   |
| KIAA1549L.Mask1.singleton   | 8.63E-05   |
| SYTL2.Mask1.0.01            | 9.26E-05   |
| POLA2.Mask1.0.01            | 0.00010398 |
| DTX3L.Mask1.0.01            | 0.00011631 |
| DSPP.Mask1.singleton        | 0.00013719 |
| FGGY.Mask1.0.01             | 0.00014164 |
| PPY.Mask1.0.01              | 0.00015847 |

### Next steps
- Rerun tests by ethnicity
- Characterize test results:
    - Produce QQ plots for each group
    - Count of how many individuals had a non-zero amount of variants for that gene
    - Distribution of variant counts per gene- Compare results with Bryan's and genebass's
- Inline PCA computation based on Broad's RV analysis notebook
- Run test on full dataset (100k + 10k)

[Discuss](https://github.com/orgs/va-big-data-genomics/discussions/29)
