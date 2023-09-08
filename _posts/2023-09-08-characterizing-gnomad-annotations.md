This post is a continuation of the characterization work I've been doing on our
data and the publicly available annotations databases. This was motivated by
the unexpected results we were seeing in our burden testing – they did not match
Genebass's results, nor Bryan Gorman's. The post is a bit code-heavy, but
hopefully easy to grasp. I've tried to write each snippet so that it will run
independently of the others; the logic in each snippet is likewise self-contained.

discuss

# Before annotating:

**What does the slimmed-down table look like?**

```
# Initialization
import hail as hl
hl.init()

# Load genomes table (rows - variant loci and alleles, cols - samples, entries - genotypes)
genomes_mt_path = 'gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k_gt.mt'
genomes_mt = hl.read_matrix_table( f'gs://{genomes_mt_path}' )
genomes_mt.describe()

# ----------------------------------------
# Column fields:
#     's': str
# ----------------------------------------
# Row fields:
#     'locus': locus<GRCh38>
#     'alleles': array<str>
# ----------------------------------------
# Entry fields:
#     'GT': call
# ----------------------------------------
# Column key: ['s']
# Row key: ['locus', 'alleles']
# ----------------------------------------

genomes_mt.count()
# (663351127, 104923)
```

**How many locus + allele pairs are there total?**

```
genomes_mt.distinct_by_row().count()
# 663,351,127 locus + allele pairs
#     104,923 sample genomes
```

**How many alleles are there per chromosome?**

```
>>> alleles_by_chr = genomes_mt.group_rows_by( contig = genomes_mt.locus.contig ).aggregate( n_alleles = hl.agg.count())
+---------+-------------------------+-------------------------+-------------------------+
| contig  | 'SHIP5297471'.n_alleles | 'SHIP5508159'.n_alleles | 'SHIP5466675'.n_alleles |
+---------+-------------------------+-------------------------+-------------------------+
| "chr1"  |                50052394 |                50063518 |                50134425 |
| "chr2"  |                54959918 |                54984604 |                55044193 |
| "chr3"  |                45141238 |                45160071 |                45215510 |

>>> alleles_by_chr = alleles_by_chr.annotate_cols( grp = 1 )
>>> alleles_by_chr_stats = alleles_by_chr.group_cols_by('grp').aggregate( mean_alleles = hl.agg.mean( alleles_by_chr.n_alleles )
+---------+----------------+
| contig  | 1.mean_alleles |
+---------+----------------+
| str     |        float64 |
+---------+----------------+
| "chr1"  |       5.01e+07 |
| "chr2"  |       5.50e+07 |
| "chr3"  |       4.52e+07 |
| "chr4"  |       4.40e+07 |
| "chr5"  |       4.07e+07 |
| "chr6"  |       3.84e+07 |
| "chr7"  |       3.58e+07 |
| "chr8"  |       3.52e+07 |
| "chr9"  |       2.82e+07 |
| "chr10" |       2.99e+07 |
| "chr11" |       3.07e+07 |
| "chr12" |       2.96e+07 |
| "chr13" |       2.20e+07 |
| "chr14" |       2.02e+07 |
| "chr15" |       1.86e+07 |
| "chr16" |       2.05e+07 |
| "chr17" |       1.78e+07 |
| "chr18" |       1.72e+07 |
| "chr19" |       1.35e+07 |
| "chr20" |       1.42e+07 |
| "chr21" |       8.39e+06 |
| "chr22" |       8.70e+06 |
| "chrX"  |       3.17e+07 |
| "chrY"  |       3.11e+06 |
+---------+----------------+
```

**How many alleles are there per sample? (average count of non-missing alleles per sample)**
```
>>> final_mt_path = 'gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k_QCed_final.MT'
>>> final_mt = hl.read_matrix_table( f'gs://{final_mt_path}' )
>>> final_mt.describe()
----------------------------------------
Column fields:
    's': str
    'sex': str
    ...
    'sample_qc': struct {
        ...
        call_rate: float64, 
        n_called: int64, 
        n_not_called: int64, 
        ...
    }
...
----------------------------------------
Row fields:
...
----------------------------------------
Entry fields:
...
----------------------------------------
Column key: ['s']
Row key: ['locus', 'alleles']
----------------------------------------

>>> final_mt.aggregate_cols( hl.agg.mean( final_mt.sample_qc.n_called ))
658575937.3369519 # 658,575,937
>>> final_mt.aggregate_cols( hl.agg.mean( final_mt.sample_qc.call_rate ))
0.992801414712833 # 99.3%
```

**How many distinct variant loci are there?**
```
genomes_mt.key_rows_by('locus').select_rows().select_entries().distinct_by_row().count())
598135035 # 598,135,035
```

# Genebass

**What does the MatrixTable look like?**
```
import hail as hl
hl.init()
genebass_mt_path   = 'ukbb-exome-public/500k/results/results.mt'
genebass_mt = hl.read_matrix_table( f'gs://{genebass_mt_path}' )

genebass_mt.describe()

----------------------------------------
Global fields:
    None
----------------------------------------
Column fields:
    'n_cases': int32
    'n_controls': int32
    'heritability': float64
    'saige_version': str
    'inv_normalized': str
    'trait_type': str
    'phenocode': str
    'pheno_sex': str
    'coding': str
    'modifier': str
    'n_cases_defined': int64
    'n_cases_both_sexes': int64
    'n_cases_females': int64
    'n_cases_males': int64
    'description': str
    'description_more': str
    'coding_description': str
    'category': str
----------------------------------------
Row fields:
    'gene_id': str
    'gene_symbol': str
    'annotation': str
    'interval': interval<locus<GRCh38>>
    'markerIDs': str
    'markerAFs': str
    'total_variants': int32
    'Nmarker_MACCate_1': int32
    'Nmarker_MACCate_2': int32
    'Nmarker_MACCate_3': int32
    'Nmarker_MACCate_4': int32
    'Nmarker_MACCate_5': int32
    'Nmarker_MACCate_6': int32
    'Nmarker_MACCate_7': int32
    'Nmarker_MACCate_8': int32
----------------------------------------
Entry fields:
    'Pvalue': float64
    'Pvalue_Burden': float64
    'Pvalue_SKAT': float64
    'BETA_Burden': float64
    'SE_Burden': float64
    'Pvalue.NA': float64
    'Pvalue_Burden.NA': float64
    'Pvalue_SKAT.NA': float64
    'BETA_Burden.NA': float64
    'SE_Burden.NA': float64
    'total_variants_pheno': int32
----------------------------------------
Column key: ['trait_type', 'phenocode', 'pheno_sex', 'coding', 'modifier']
Row key: ['gene_id', 'gene_symbol', 'annotation']
----------------------------------------

genebass_mt.count()
# (75767, 4529)
```

**How many genes are in Genebass?**
```
import hail as hl
hl.init()
genebass_mt_path   = 'ukbb-exome-public/500k/results/results.mt'
genebass_mt = hl.read_matrix_table( f'gs://{genebass_mt_path}' )
genebass_mt.key_rows_by('gene_symbol').distinct_by_row().count()
# 19,403 genes
#  4,529 phenotypes
```
So there are 75,767 rows for 19,403 genes, giving us 3.9 annotations per gene, on average.

**How many different annotations are possible?**

```
genebass_mt.key_rows_by('annotation').distinct_by_row().count()
# 4
```

Which means that almost every gene has the full complement of annotations. This makes sense if some variants in a gene are pLoF, others are mis-sense, still others are long-coding, and so on.

**How many distinct variant loci are in Genebass?**

```
import hail as hl
hl.init()
genebass_mt_path   = 'ukbb-exome-public/500k/results/results.mt'
genebass_mt = hl.read_matrix_table( f'gs://{genebass_mt_path}' )
genebass_mt = genebass_mt.key_rows_by( 'interval' )
genebass_mt = genebass_mt.key_cols_by( 'phenocode' )
genebass_mt = genebass_mt.transmute_rows( variants = genebass_mt.markerIDs.split(';'))
genebass_mt = genebass_mt.explode_rows( 'variants' )
genebass_mt = genebass_mt.transmute_rows( variant = genebass_mt.variants.replace('[_/]',':'))
genebass_mt = genebass_mt.transmute_rows( variant = hl.parse_variant( genebass_mt.variant, reference_genome='GRCh38' ))
genebass_mt = genebass_mt.transmute_rows( locus = genebass_mt.variant.locus, alleles = genebass_mt.variant.alleles )
genebass_mt.key_rows_by('locus').distinct_by_row().count()
# 7,173,997 loci
#     4,529 phenotypes
```

**How many alleles are in Genebass?**
```
import hail as hl
hl.init()
genebass_mt_path   = 'ukbb-exome-public/500k/results/results.mt'
genebass_mt = hl.read_matrix_table( f'gs://{genebass_mt_path}' )
genebass_mt = genebass_mt.key_rows_by( 'interval' )
genebass_mt = genebass_mt.key_cols_by( 'phenocode' )
genebass_mt = genebass_mt.transmute_rows( variants = genebass_mt.markerIDs.split(';'))
genebass_mt = genebass_mt.explode_rows( 'variants' )
genebass_mt = genebass_mt.transmute_rows( variant = genebass_mt.variants.replace('[_/]',':'))
genebass_mt = genebass_mt.transmute_rows( variant = hl.parse_variant( genebass_mt.variant, reference_genome='GRCh38' ))
genebass_mt = genebass_mt.transmute_rows( locus = genebass_mt.variant.locus, alleles = genebass_mt.variant.alleles )
genebass_mt.key_rows_by('locus','alleles').distinct_by_row().count()
# 7,961,375 alleles
#     4,529 phenotypes
```

This is ~1.1 alleles per locus.

**How many variants of each annotation type are in Genebass?**
```
genebass_mt_path = 'ukbb-exome-public/500k/results/results.mt'
genebass_mt = hl.read_matrix_table( f'gs://{genebass_mt_path}' )
genebass_mt = genebass_mt.key_rows_by( 'interval' )
genebass_mt = genebass_mt.key_cols_by( 'phenocode' )
genebass_mt = genebass_mt.transmute_rows( variants = genebass_mt.markerIDs.split(';'))
genebass_mt = genebass_mt.explode_rows( 'variants' )
genebass_mt = genebass_mt.transmute_rows( variant = genebass_mt.variants.replace('[_/]',':'))
genebass_mt = genebass_mt.transmute_rows( variant = hl.parse_variant( genebass_mt.variant, reference_genome='GRCh38' ))
genebass_mt = genebass_mt.transmute_rows( locus = genebass_mt.variant.locus, alleles = genebass_mt.variant.alleles )
genebass_mt = genebass_mt.key_rows_by('locus','alleles')
genebass_mt = genebass_mt.select_rows('annotation').select_cols().select_entries()

genebass_mt.aggregate_rows( hl.agg.counter( genebass_mt.annotation ))
#| annotation       | average   |
#|------------------|-----------|
#| missense^LC      | 5,300,968 |
#| pLoF             |   521,821 |
#| pLoF^missense^LC | 5,692,889 |
#| synonymous       | 2,272,251 |
```

## After annotating our genomes with Genebass's gene symbols

```
# Initialization
import hail as hl
hl.init()

# Get Genebass's results MatrixTable
genebass_mt_path   = 'ukbb-exome-public/500k/results/results.mt'
genebass_mt = hl.read_matrix_table( f'gs://{genebass_mt_path}' )

# Sample row data from Genebass (note the delimited `annotation` and `markerIDs` fields):
# | gene_id           | gene_symbol | interval                        | annotation         | markerIDs                                                                                                                        |
# |-------------------|-------------|---------------------------------|--------------------|----------------------------------------------------------------------------------------------------------------------------------|
# | "ENSG00000157766" | "ACAN"      | [chr15:88836159-chr15:88880352) | "missense|LC"      | "chr15:88836207_A/G;chr15:88836211_C/T;chr15:88836217_T/C;chr15:88836219_C/T;chr15:88836231_G/A;chr15:88836235_C/T;chr15:888..." |
# | "ENSG00000157766" | "ACAN"      | [chr15:88836159-chr15:88880352) | "pLoF"             | "chr15:88838945_TG/T;chr15:88838956_AG/A;chr15:88840177_AG/A;chr15:88843650_T/C;chr15:88845771_A/AC;chr15:88847364_C/G;chr15..." |
# | "ENSG00000157766" | "ACAN"      | [chr15:88836159-chr15:88880352) | "pLoF|missense|LC" | "chr15:88836207_A/G;chr15:88836211_C/T;chr15:88836217_T/C;chr15:88836219_C/T;chr15:88836231_G/A;chr15:88836235_C/T;chr15:888..." |
# | "ENSG00000157766" | "ACAN"      | [chr15:88836159-chr15:88880352) | "synonymous"       | "chr15:88836230_C/T;chr15:88838667_T/C;chr15:88838676_G/A;chr15:88838677_C/T;chr15:88838679_G/A;chr15:88838682_T/C;chr15:888..." |

# Get the pLoF variants' genes as reported in Genebass
genebass_mt = genebass_mt.key_rows_by( 'interval' )
genebass_mt = genebass_mt.key_cols_by( 'phenocode' )

# Unpack variant structure to match ours (one variant's locus and one pair of its reference and alternate alleles per row)
genebass_mt = genebass_mt.transmute_rows( variants = genebass_mt.markerIDs.split(';'))
genebass_mt = genebass_mt.explode_rows( 'variants' )
genebass_mt = genebass_mt.transmute_rows( variant = genebass_mt.variants.replace('[_/]',':'))
genebass_mt = genebass_mt.transmute_rows( variant = hl.parse_variant( genebass_mt.variant, reference_genome='GRCh38' ))
genebass_mt = genebass_mt.transmute_rows( locus = genebass_mt.variant.locus, alleles = genebass_mt.variant.alleles )
genebass_mt = genebass_mt.key_rows_by( 'locus', 'alleles' )

# Filter and select MatrixTable to needed rows/cols/entries only
genebass_mt = genebass_mt.filter_rows( genebass_mt.annotation == 'pLoF' )
genebass_mt = genebass_mt.filter_cols( genebass_mt.phenocode == 'height_custom' )
genebass_mt = genebass_mt.select_rows( 'gene_id','gene_symbol' ) # 'interval'
genebass_mt = genebass_mt.select_cols( )
genebass_mt = genebass_mt.select_entries( ) # 'Pvalue_Burden'

# genebass_mt.head(5).entries().show()
# | gene_id           | gene_symbol | locus         | alleles       | phenocode       |
# |-------------------|-------------|---------------|---------------|-----------------|
# | "ENSG00000187634" | "SAMD11"    | chr1:925960   | ["C","CAGGT"] | "height_custom" |
# | "ENSG00000187634" | "SAMD11"    | chr1:925960   | ["C","T"]     | "height_custom" |
# | "ENSG00000187634" | "SAMD11"    | chr1:925996   | ["C","T"]     | "height_custom" |
# | "ENSG00000187634" | "SAMD11"    | chr1:926004   | ["C","CGA"]   | "height_custom" |
# | "ENSG00000187634" | "SAMD11"    | chr1:926005   | ["TC","T"]    | "height_custom" |

# Load genomes table (rows - variant loci and alleles, cols - samples, entries - genotypes)
genomes_mt_path = 'gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k_gt.mt'
genomes_mt = hl.read_matrix_table( f'gs://{genomes_mt_path}' )

# Annotate our variants with Genebass's gene symbols
genomes_mt = genomes_mt.annotate_rows( gene = genebass_mt.rows()[ genomes_mt.locus, genomes_mt.alleles ].gene_symbol )
```

**How many matching genes?**
```
genomes_mt.filter_rows( hl.is_defined( genomes_mt.gene)).key_rows_by('gene').distinct_by_row().count()
# 15,769 vs 19,403 genes in Genebass
```

**How many matching loci within those genes?**
```
genomes_mt.filter_rows( hl.is_defined( genomes_mt.gene )).key_rows_by('locus').distinct_by_row().count()
# 121,179
```

**How many matching alleles at those loci?**
```
genomes_mt.filter_rows( hl.is_defined( genomes_mt.gene )).key_rows_by('locus','alleles').distinct_by_row().count()
# 123,175 loci + allele combinations
```

So 123,175 / 663,351,127 = 0.0186% of the variants in the MVP genomes dataset are correlated with height in Genebass's annotations database.

**How many misses?**
```
genomes_mt.filter_rows( hl.is_missing( genomes_mt.gene )).key_rows_by('locus','alleles').distinct_by_row().count()
# 663,227,952 loci + allele combinations
```
This seems like a lot of misses, until you recall that we are filtering for a single phenotype (height) and a single annotation (pLoF).

So for 104,923 samples, we found 123,175 putative loss of function alleles at 121,179 loci in 15,769 genes, that were correlated to some degree with height.

**What proportion of variants in the MVP genomes are represented in Genebass?**
```
genebass_mt_path = 'ukbb-exome-public/500k/results/results.mt'
genebass_mt = hl.read_matrix_table( f'gs://{genebass_mt_path}' )

genomes_mt_path = 'gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k_gt.mt'
genomes_mt = hl.read_matrix_table( f'gs://{genomes_mt_path}' )

genebass_mt = genebass_mt.key_rows_by('interval')
genebass_matches_mt = genomes_mt.filter_rows( hl.is_defined( genebass_mt.rows()[ genomes_mt.locus ]))
# 234,358,589 (35.3% of MVP's total variants)
```

**How many of the pLoF variants in Genebass are found in the MVP genomes?**
```
# Join Genebass's variants to ours to see how many match
genomes_mt_path = 'gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k_gt.mt'
genomes_mt = hl.read_matrix_table( f'gs://{genomes_mt_path}' )

genebass_mt.filter_rows( genebass_mt.annotation == 'pLoF').rows().join( genomes_mt.rows() ).count()
# 124,888 (23.9% of Genebass's 522k total)
```

# gnomAD

**What does the MatrixTable look like?**
```
import hail as hl
hl.init()
gnomad_ht_path = 'gcp-public-data--gnomad/release/3.1.2/ht/genomes/gnomad.genomes.v3.1.2.sites.ht'
gnomad_ht = hl.read_table( f'gs://{gnomad_ht_path}' )

gnomad_ht.describe()
----------------------------------------
Global fields:
    'freq_meta': array<dict<str, str>> 
    'freq_index_dict': dict<str, int32> 
    'faf_index_dict': dict<str, int32> 
    'faf_meta': array<dict<str, str>> 
    'vep_version': str 
    'vep_csq_header': str 
    'dbsnp_version': str 
    'filtering_model': struct {
        model_name: str, 
        score_name: str, 
        snv_cutoff: struct {
            bin: float64, 
            min_score: float64
        }, 
        indel_cutoff: struct {
            bin: float64, 
            min_score: float64
        }, 
        model_id: str, 
        snv_training_variables: array<str>, 
        indel_training_variables: array<str>
    } 
    'age_distribution': struct {
        bin_edges: array<float64>, 
        bin_freq: array<int32>, 
        n_smaller: int32, 
        n_larger: int32
    } 
    'freq_sample_count': array<int32> 
----------------------------------------
Row fields:
    'locus': locus<GRCh38> 
    'alleles': array<str> 
    'freq': array<struct {
        AC: int32, 
        AF: float64, 
        AN: int32, 
        homozygote_count: int32
    }> 
    'raw_qual_hists': struct {
        gq_hist_all: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        dp_hist_all: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        gq_hist_alt: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        dp_hist_alt: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        ab_hist_alt: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }
    } 
    'popmax': struct {
        AC: int32, 
        AF: float64, 
        AN: int32, 
        homozygote_count: int32, 
        pop: str, 
        faf95: float64
    } 
    'qual_hists': struct {
        gq_hist_all: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        dp_hist_all: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        gq_hist_alt: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        dp_hist_alt: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }, 
        ab_hist_alt: struct {
            bin_edges: array<float64>, 
            bin_freq: array<int64>, 
            n_smaller: int64, 
            n_larger: int64
        }
    } 
    'faf': array<struct {
        faf95: float64, 
        faf99: float64
    }> 
    'a_index': int32 
    'was_split': bool 
    'rsid': set<str> 
    'filters': set<str> 
    'info': struct {
        QUALapprox: int64, 
        SB: array<int32>, 
        MQ: float64, 
        MQRankSum: float64, 
        VarDP: int32, 
        AS_ReadPosRankSum: float64, 
        AS_pab_max: float64, 
        AS_QD: float32, 
        AS_MQ: float64, 
        QD: float32, 
        AS_MQRankSum: float64, 
        FS: float64, 
        AS_FS: float64, 
        ReadPosRankSum: float64, 
        AS_QUALapprox: int64, 
        AS_SB_TABLE: array<int32>, 
        AS_VarDP: int32, 
        AS_SOR: float64, 
        SOR: float64, 
        singleton: bool, 
        transmitted_singleton: bool, 
        omni: bool, 
        mills: bool, 
        monoallelic: bool, 
        AS_VQSLOD: float64, 
        InbreedingCoeff: float64
    } 
    'vep': struct {
        assembly_name: str, 
        allele_string: str, 
        ancestral: str, 
        context: str, 
        end: int32, 
        id: str, 
        input: str, 
        intergenic_consequences: array<struct {
            allele_num: int32, 
            consequence_terms: array<str>, 
            impact: str, 
            minimised: int32, 
            variant_allele: str
        }>, 
        most_severe_consequence: str, 
        motif_feature_consequences: array<struct {
            allele_num: int32, 
            consequence_terms: array<str>, 
            high_inf_pos: str, 
            impact: str, 
            minimised: int32, 
            motif_feature_id: str, 
            motif_name: str, 
            motif_pos: int32, 
            motif_score_change: float64, 
            strand: int32, 
            variant_allele: str
        }>, 
        regulatory_feature_consequences: array<struct {
            allele_num: int32, 
            biotype: str, 
            consequence_terms: array<str>, 
            impact: str, 
            minimised: int32, 
            regulatory_feature_id: str, 
            variant_allele: str
        }>, 
        seq_region_name: str, 
        start: int32, 
        strand: int32, 
        transcript_consequences: array<struct {
            allele_num: int32, 
            amino_acids: str, 
            appris: str, 
            biotype: str, 
            canonical: int32, 
            ccds: str, 
            cdna_start: int32, 
            cdna_end: int32, 
            cds_end: int32, 
            cds_start: int32, 
            codons: str, 
            consequence_terms: array<str>, 
            distance: int32, 
            domains: array<struct {
                db: str, 
                name: str
            }>, 
            exon: str, 
            gene_id: str, 
            gene_pheno: int32, 
            gene_symbol: str, 
            gene_symbol_source: str, 
            hgnc_id: str, 
            hgvsc: str, 
            hgvsp: str, 
            hgvs_offset: int32, 
            impact: str, 
            intron: str, 
            lof: str, 
            lof_flags: str, 
            lof_filter: str, 
            lof_info: str, 
            minimised: int32, 
            polyphen_prediction: str, 
            polyphen_score: float64, 
            protein_end: int32, 
            protein_start: int32, 
            protein_id: str, 
            sift_prediction: str, 
            sift_score: float64, 
            strand: int32, 
            swissprot: str, 
            transcript_id: str, 
            trembl: str, 
            tsl: int32, 
            uniparc: str, 
            variant_allele: str
        }>, 
        variant_class: str
    } 
    'vqsr': struct {
        AS_VQSLOD: float64, 
        AS_culprit: str, 
        NEGATIVE_TRAIN_SITE: bool, 
        POSITIVE_TRAIN_SITE: bool
    } 
    'region_flag': struct {
        lcr: bool, 
        segdup: bool
    } 
    'allele_info': struct {
        variant_type: str, 
        allele_type: str, 
        n_alt_alleles: int32, 
        was_mixed: bool
    } 
    'age_hist_het': struct {
        bin_edges: array<float64>, 
        bin_freq: array<int64>, 
        n_smaller: int64, 
        n_larger: int64
    } 
    'age_hist_hom': struct {
        bin_edges: array<float64>, 
        bin_freq: array<int64>, 
        n_smaller: int64, 
        n_larger: int64
    } 
    'cadd': struct {
        phred: float32, 
        raw_score: float32, 
        has_duplicate: bool
    } 
    'revel': struct {
        revel_score: float64, 
        has_duplicate: bool
    } 
    'splice_ai': struct {
        splice_ai_score: float32, 
        splice_consequence: str, 
        has_duplicate: bool
    } 
    'primate_ai': struct {
        primate_ai_score: float32, 
        has_duplicate: bool
    } 
----------------------------------------
Key: ['locus', 'alleles']
----------------------------------------

gnomad_ht.count()
# 759,302,267
```

**Sample data**
```
| locus      | alleles                | vep.transcript_consequences.gene_id                                          | vep.transcript_consequences.gene_symbol           | vep.most_severe_consequence |
|------------|------------------------|------------------------------------------------------------------------------|---------------------------------------------------|-----------------------------|
| chr1:10031 | ["T","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10037 | ["T","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10043 | ["T","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10055 | ["T","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10057 | ["A","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10061 | ["T","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10061 | ["T","TAACCCTAACC..."] | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10064 | ["C","CCCTAACCCTA..."] | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10067 | ["T","TAACCCTAACC..."] | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10108 | ["C","CA"]             | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10108 | ["C","CAACCCTAACC..."] | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10108 | ["CAACCCT","C"]        | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10109 | ["A","T"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10109 | ["AACCCT","A"]         | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10111 | ["C","A"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10113 | ["CT","C"]             | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10114 | ["T","A"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10114 | ["T","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10114 | ["TA","T"]             | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10114 | ["TAACCCTAACCCTAA..."] | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10116 | ["A","G"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10116 | ["AC","A"]             | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10116 | ["ACCCTAACCCTAACC..."] | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10117 | ["CCCTAA","C"]         | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10118 | ["C","T"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10119 | ["CT","C"]             | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10120 | ["T","A"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10120 | ["T","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10120 | ["T","G"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10120 | ["TA","T"]             | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10122 | ["A","C"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |
| chr1:10122 | ["A","G"]              | ["ENSG00000223972","ENSG00000223972","ENSG00000227232","653635","100287102"] | ["DDX11L1","DDX11L1","WASH7P","WASH7P","DDX11L1"] | "upstream_gene_variant"     |

```

**How many genes?**
```
import hail as hl
hl.init()
gnomad_ht_path = 'gcp-public-data--gnomad/release/3.1.2/ht/genomes/gnomad.genomes.v3.1.2.sites.ht'
gnomad_ht = hl.read_table( f'gs://{gnomad_ht_path}' )
gnomad_ht_anno_gs = gnomad_ht.select( gene_symbol = gnomad_ht.vep.transcript_consequences.gene_symbol )
gnomad_ht_anno_gs_exploded = gnomad_ht_anno_gs.explode( gnomad_ht_anno_gs.gene_symbol )
gnomad_ht_anno_gs_exploded.key_by('gene_symbol').distinct().count()
# 71,966
```

This is almost _4 times the number of genes in Genebass_, and if there are only supposed to be ~20-25k genes in the human genome, this must mean gnomAD is including a number of other genes - pseudogenes, perhaps?

**How many different annotations are possible?**
```
import hail as hl
hl.init()
gnomad_ht_path = 'gcp-public-data--gnomad/release/3.1.2/ht/genomes/gnomad.genomes.v3.1.2.sites.ht'
gnomad_ht = hl.read_table( f'gs://{gnomad_ht_path}' )
# 30
```

**How many variants of each annotation type are in gnomAD?**
```
import hail as hl
hl.init()
gnomad_ht_path = 'gcp-public-data--gnomad/release/3.1.2/ht/genomes/gnomad.genomes.v3.1.2.sites.ht'
gnomad_ht = hl.read_table( f'gs://{gnomad_ht_path}' )
gnomad_ht.aggregate( hl.agg.counter( gnomad_ht.vep.most_severe_consequence ))
gnomad_ht.key_by( gnomad_ht.vep.most_severe_consequence ).distinct().count()
```

After some reformatting for Markdown and sorting largest to smallest:
| consequence                        | n         |
|------------------------------------|-----------|
| intron_variant                     | 446039246 |
| intergenic_variant                 | 183401726 |
| upstream_gene_variant              | 34666021  |
| downstream_gene_variant            | 27860717  |
| regulatory_region_variant          | 20343492  |
| non_coding_transcript_exon_variant | 19951729  |
| 3_prime_UTR_variant                | 11292406  |
| missense_variant                   | 5020199   |
| 5_prime_UTR_variant                | 4619937   |
| synonymous_variant                 | 2305362   |
| TF_binding_site_variant            | 1606770   |
| splice_region_variant              | 1194398   |
| frameshift_variant                 | 300714    |
| stop_gained                        | 173771    |
| splice_donor_variant               | 146188    |
| splice_acceptor_variant            | 135144    |
| inframe_deletion                   | 98366     |
| inframe_insertion                  | 60238     |
| start_lost                         | 28770     |
| TFBS_ablation                      | 16727     |
| stop_lost                          | 15576     |
| mature_miRNA_variant               | 13979     |
| stop_retained_variant              | 4839      |
| protein_altering_variant           | 4416      |
| coding_sequence_variant            | 868       |
| incomplete_terminal_codon_variant  | 484       |
| transcript_ablation                | 107       |
| regulatory_region_ablation         | 55        |
| non_coding_transcript_variant      | 18        |
| start_retained_variant             | 4         |

The four we're using are:
| most severe consequence            | variant count |
|------------------------------------|---------------|
| start_lost                         |  28770        |
| stop_gained                        | 173771        |
| splice_donor_variant               | 146188        |
| splice_acceptor_variant            | 135144        |
| total                              | 483873        |

So we're looking at 483,873 / 759,302,267 = **0.0637%** of the total number of variants.

**How many distinct variant loci are in gnomAD?**
```
import hail as hl
hl.init()
gnomad_ht_path = 'gcp-public-data--gnomad/release/3.1.2/ht/genomes/gnomad.genomes.v3.1.2.sites.ht'
gnomad_ht = hl.read_table( f'gs://{gnomad_ht_path}' )
gnomad_ht.key_by('locus').distinct().count()
# 629,971,803
```

**How many distinct alleles are in gnomAD?**
```
import hail as hl
hl.init()
gnomad_ht_path = 'gcp-public-data--gnomad/release/3.1.2/ht/genomes/gnomad.genomes.v3.1.2.sites.ht'
gnomad_ht = hl.read_table( f'gs://{gnomad_ht_path}' )
gnomad_ht.key_by('locus','alleles').distinct().count()
# 759,302,267
```

This is ~1.2 alleles per locus.

## After annotating our genomes with gnomAD's gene symbols

```
import hail as hl
hl.init()

# Import gnomAD annotations database
gnomad_ht_path = 'gcp-public-data--gnomad/release/3.1.2/ht/genomes/gnomad.genomes.v3.1.2.sites.ht'
gnomad_ht = hl.read_table( f'gs://{gnomad_ht_path}' )

# Filter & select needed fields from gnomAD
gnomad_ht = gnomad_ht.filter( \
  (gnomad_ht.vep.most_severe_consequence == 'start_lost') |\
  (gnomad_ht.vep.most_severe_consequence == 'stop_gained') |\
  (gnomad_ht.vep.most_severe_consequence == 'splice_donor_variant') |\
  (gnomad_ht.vep.most_severe_consequence == 'splice_acceptor_variant'))
gnomad_ht = gnomad_ht.select( gnomad_ht.vep.transcript_consequences, gnomad_ht.vep.most_severe_consequence )
gnomad_ht = gnomad_ht.explode('transcript_consequences')
gnomad_ht = gnomad_ht.transmute( gene_id = gnomad_ht.transcript_consequences.gene_id, \
                                 gene_symbol = gnomad_ht.transcript_consequences.gene_symbol, \
                                 most_severe_consequence = gnomad_ht.most_severe_consequence )
gnomad_ht = gnomad_ht.select_globals()

# Load MVP genomes table (rows - variant loci and alleles, cols - samples, entries - genotypes)
genomes_mt_path = 'gbsc-gcp-project-mvp-wgs-data-release-2/gvcf_aggregation_100k/release_20230505/rel2_100k_gt.mt'
genomes_mt = hl.read_matrix_table( f'gs://{genomes_mt_path}' )

# Annotate our variants with gnomAD's gene symbols
genomes_mt = genomes_mt.annotate_rows( gene = gnomad_ht[ genomes_mt.locus, genomes_mt.alleles ].gene_symbol )
```

**How many matching genes?**
```
genomes_mt.filter_rows( hl.is_defined( genomes_mt.gene)).key_rows_by('gene').distinct_by_row().count()
# 28,229 genes
# 104,923 samples
```

**How many matching loci within those genes?**
```
genomes_mt.filter_rows( hl.is_defined( genomes_mt.gene )).key_rows_by('locus').distinct_by_row().count()
# 201,973 loci
```

So ~7 variant loci per gene matching our genomes.

**How many matching alleles at those loci?**
```
genomes_mt.filter_rows( hl.is_defined( genomes_mt.gene )).key_rows_by('locus','alleles').distinct_by_row().count()
# 209,188
```

So ~1 allele per locus.

**How many misses?**
```
genomes_mt.filter_rows( hl.is_missing( genomes_mt.gene )).key_rows_by('locus','alleles').distinct_by_row().count()

```

**What proportion of variants in the MVP genomes are represented in gnomAD?**
```
TBD
```

**How many of the "pLoF" variants in gnomAD are found in the MVP genomes?**
```
TBD
```
