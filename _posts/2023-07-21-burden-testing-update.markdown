---
layout: post
title:  "#27 Burden testing progress update 2"
date:   2023-07-21 12:30:00 -0800
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
genomes_mt = genomes_mt.filter_rows( hl.is_missing( genomes_mt.gene ), keep=False )
genomes_mt = genomes_mt.select_rows( genomes_mt.gene )
genomes_mt = genomes_mt.select_cols( genomes_mt.super_pop )
genomes_mt = genomes_mt.select_entries( genomes_mt.GT )
```

Shell:
```
% gsutil du -s -h gs://gbsc-gcp-project-mvp-wgs-data-release-2/burden-testing/genomes-10k-slim.mt
4.24 MiB     gs://gbsc-gcp-project-mvp-wgs-data-release-2/burden-testing/genomes-10k-slim.mt
```

### Results


