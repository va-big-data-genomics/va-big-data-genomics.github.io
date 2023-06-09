---
layout: post
title:  "#23 Burden testing progress update"
date:   2023-02-03 10:11:02 -0800
author: Daniel Cotter 
categories: jekyll update
---

I was hoping to have burden testing results for this post, but I ran into a number of hurdles along the way and am still working towards the eventual goal of matching genebass and Bryan's results. However, the hurdles themselves tell an interesting story and provide some insight into how best to use Hail and GCP, so I polished up my working log thus far and will share the results when I have them. Enjoy!

### The goal

My first attempt at burden testing fell flat, in that it didn't match [genebass's height-associated genes](https://app.genebass.org/gene/undefined/phenotype/continuous-height_male_custom-males--custom) or Bryan's (which matched each other, by the way). For the second attempt, I wanted to restrict my testing to the most deleterious, or"putative loss of function (pLoF)," variants. Basing the list of putative/predicted loss-of-function variants on [The mutational constraint spectrum... (Karczewski et al., 2020)](https://www.nature.com/articles/s41586-020-2308-7) and the associated software package's [documentation](https://github.com/konradjk/loftee#readme), I came up with the following list of variant types:

- Stop-gained
- Splice site disrupting
- Frameshift variants

### Hurdle #1: Backwards incompatibility

Hail includes logic to download and use annotation databases, with [variants listed by consequence](https://www.hail.is/docs/0.1/annotationdb.html#gene-level-annotations); however, when I started working on this revision, I was looking at [v0.1 of the annotation database docs](https://www.hail.is/docs/0.1/annotationdb.html#database-query). Version 0.2 has since been released, and unfortunately, the code given as an example for v0.1 no longer works:

```
import hail
from pprint import pprint

hc = hail.HailContext()
#         ^^^ no longer exists in v0.2!!
vds = (
    hc
    .import_vcf('gs://annotationdb/test/sample.vcf')
    .split_multi()
    .annotate_variants_db([
        'va.cadd'
    ])
)

pprint(vds.variant_schema)
```

### Hurdle #2: Too many choices

Version 0.2 of Hail includes the same functionality, but the [list of available databases](https://hail.is/docs/0.2/annotation_database_ui.html#database-query) is overwhelming in its size. After talking to Paul, it seemed the [gnomad\_genome\_sites](https://hail.is/docs/0.2/datasets/schemas/gnomad_genome_sites.html) was a good place to start.

### Hurdle #3: VERY big data

When I spun up the cluster and started to experiment with the new annotations API, the command to download the genomes dataset was taking a long time to return. I wanted to know how big these datasets were, but I couldn't just look at the size in the Storage UI, because Hail Matrix Tables are not stored as single files; they are stored as directory trees with metadata files, subdirectories for columns, rows, and entries, hundreds of data partitions, and so on. To get around this, I opened another tab and used Google Storage's "disk usage" command to see how big the datasets were while I was waiting. `gsutil du -s -h gs://<bucket>/<prefix>` returns a sum of the sizes of all the individual files in a directory tree, so I used this to get an idea of the size of these datasets. Even this took several hours to return, so I made sure to record the data for quick reference:

| dataset             | size (TB)     | bucket                                  | prefix                                                                   |
| ------------------- | ------------- | --------------------------------------- | ------------------------------------------------------------------------ |
| 10k genomes         |   6.691       | gbsc-gcp-project-mvp-covid19            | gvcf_aggregation_10k/release_20211207/20211207_covid19_10k_QCed_final.MT |
| 100k genomes        | 207.656       | gbsc-gcp-project-mvp-wgs-data-release-2 | gvcf_aggregation_100k/release_20230505/rel2_100k_QCed_final.MT           |
| gnomAD genome sites |   1.846       | gcp-public-data--gnomad                 | release/3.1/ht/genomes/gnomad.genomes.v3.1.sites.ht                      |

With datasets this big, the downloads were going to take forever, and the egress charges might be substantial (the gnomAD data is "requester pays"), so I ruled them out for quick experimentation and went looking for a smaller dataset I could use to familiarize myself with the annotations API. I could also use this smaller dataset to figure out how to do the burden testing on just the pLoF variants, then when everything was working, swap in the storage location of the larger dataset.

### Hurdle #4: No such file system

Looking through the [Matrix Table tutorial](https://hail.is/docs/0.2/tutorials/07-matrixtable.html#Importing-and-Reading), I saw they were using the 1000 Genomes dataset, so I figured it must be of a manageable size (also, it was an order of magnitude smaller than the MVP dataset). The download took a minute or two to run, and when I ran `du -s` locally, it said the dataset was 72 MB – much better than 7 TB.

However, when I ran the [code](https://hail.is/docs/0.2/experimental/hail.experimental.DB.html#hail.experimental.DB.annotate_rows_db) to annotate the rows, I got an odd error – odd because on my laptop, where I was doing my testing, I use `gsutil` all the time, so I would expect the `gs` file system to work as well.

```
>>> import hail as hl
>>> hl.utils.get_1kg('data/')
>>> mt = hl.read_matrix_table('data/1kg.mt')
>>> db = hl.experimental.DB(region='us', cloud='gcp')
>>> mt = db.annotate_rows_db(mt, 'gnomad_genome_sites')
org.apache.hadoop.fs.UnsupportedFileSystemException: No FileSystem for scheme "gs"
```

I posted a question to the [Hail chat room](https://hail.zulipchat.com/#narrow/stream/123010-Hail-Query-0.2E2-support/topic/annotate_rows_db.28.29.20question), and after some back and forth, found that the annotation database won't work unless on a cluster (important information *not* in the docs). The same code ran without error on the cluster, and I could see the annotations with `mt.describe()`:

```
>>> mt.describe()
----------------------------------------
Global fields:
    None
----------------------------------------
Column fields:
    's': str
----------------------------------------
Row fields:
    'locus': locus<GRCh37>
    'alleles': array<str>
    'rsid': str
    'qual': float64
    'filters': set<str>
    'info': struct {
        AC: array<int32>, 
        AF: array<float64>, 
        AN: int32, 
        BaseQRankSum: float64, 
        ClippingRankSum: float64, 
        DP: int32, 
        DS: bool, 
        FS: float64, 
        HaplotypeScore: float64, 
        InbreedingCoeff: float64, 
        MLEAC: array<int32>, 
        MLEAF: array<float64>, 
        MQ: float64, 
        MQ0: int32, 
        MQRankSum: float64, 
        QD: float64, 
        ReadPosRankSum: float64, 
        set: str
    }
    'gnomad_genome_sites': struct {
        freq: array<struct {
            AC: int32, 
            AF: float64, 
            AN: int32, 
            homozygote_count: int32
        }>, 
        age_hist_het: array<struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }>, 
...
```

### Getting closer

Now I was getting closer. Looking through the output of `mt.gnomad_genome_sites.show()`, I found several useful row fields in the annotations:

```
gnomad_genome_sites.variant_type
gnomad_genome_sites.allele_type
gnomad_genome_sites.vep.assembly_name
gnomad_genome_sites.vep.allele_string
gnomad_genome_sites.vep.id
gnomad_genome_sites.vep.end
gnomad_genome_sites.vep.most_severe_consequence
```

The last, in particular, looked promising, with several of the terms we used appearing in the results. The variant types we were looking for, again, were:

- Stop-gained
- Splice site disrupting
- Frameshift variants

But I needed to match these plain-English types into Hail's coded vocabulary, and I am still a long way from fluent in Hail. A `distinct()` or `unique()` function would have been ideal, but a function that groups and counts distinct values would get pretty close, and I'd just seen one in some sample code:

```
>>> mt.aggregate_rows( hl.agg.counter( mt.gnomad_genome_sites.vep.most_severe_consequence ))
{'3_prime_UTR_variant': 127, '5_prime_UTR_variant': 30, 'TF_binding_site_variant': 2, 'downstream_gene_variant': 292, 'intergenic_variant': 2494, 'intron_variant': 3633, 'missense_variant': 3015, 'non_coding_transcript_exon_variant': 141, 'regulatory_region_variant': 360, 'splice_acceptor_variant': 9, 'splice_donor_variant': 10, 'splice_region_variant': 69, 'start_lost': 6, 'stop_gained': 25, 'stop_lost': 3, 'synonymous_variant': 328, 'upstream_gene_variant': 291, None: 44}
```

So "stop-gained" becomes `stop_gained`, but "splice site disrupting" and "frameshift variants" don't have an obvious match. Could this be because the 1000 Genomes dataset is small (compared to MVP's)?

### Next steps

I will run the same code on the 10k MVP dataset to see if more "most severe consequences" values are found in there, then I will use those to filter the variants used in the burden testing algorithm. Hopefully, more results next week!

[Discuss!](https://github.com/orgs/va-big-data-genomics/discussions/26)
