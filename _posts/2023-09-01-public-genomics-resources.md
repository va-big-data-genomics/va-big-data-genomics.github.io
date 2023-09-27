---
layout: post
title:  "#31 Public Genomics Resources"
date:   2023-09-01 10:00:00 -0800
author: Daniel Cotter
categories: jekyll update
---

# Resources

### All of Us Data Browser

[Browse](https://databrowser.researchallofus.org/) aggregate-level data contributed by All of Us research participants. Data are derived from multiple data sources. To protect participant privacy, we have removed personal identifiers, rounded aggregate data to counts of 20, and only included summary demographic information. Individual-level data are available for analysis in the Researcher Workbench.

### BRAVO

This version of [BRAVO variant browser](https://bravo.sph.umich.edu/freeze8/hg38/) shows chromosome locations (on GRCh38 human genome assembly), alleles, functional annotations, and allele frequencies for 705 million variants observed in 132,345 deeply sequenced (>38X) genomes from the TOPMed (Trans-Omics for Precision Medicine) data freeze 8.

### CADD

[Combined Annotation Dependent Depletion (CADD)](https://cadd.gs.washington.edu/) is a tool for scoring the deleteriousness of single nucleotide variants as well as insertion/deletions variants in the human genome. Developed by UW, Brotman-Baty, Hudson-Alpha, and Berlin Institute of Health.

### dbGaP

[Database of Genotypes and Phenotypes (dbGaP)](https://www.ncbi.nlm.nih.gov/gap/), a repository of information produced by studies investigating the interaction of genotype and phenotype.

### dbSNP

[dbSNP](https://www.ncbi.nlm.nih.gov/projects/SNP/snp_summary.cgi) is a single-nucleotide-polymorphism database - a free public archive for genetic variation within and across different species developed and hosted by the National Center for Biotechnology Information (NCBI) in collaboration with the National Human Genome Research Institute (NHGRI).
 
### Ensembl

[Ensembl](https://useast.ensembl.org/index.html) provides an automatic gene annotation for Homo sapiens. In the case of human and mouse, the GTF files are equivalent to the GENCODE gene set. Developed by the European Bioinformatics Institute, under the European Molecular Biology Laboratory.

### Genebass

[Genebass ("gene-based association summary statistics")](https://app.genebass.org/) is a **resource** of exome-based association statistics, made available to the public. The dataset encompasses 4,529 phenotypes with gene-based and single-variant testing across 394,841 individuals with exome sequence data from the UK Biobank. Genebass was developed by the following **organizations** which provided funding and guidance: AbbVie, Biogen, Pfizer, Broad Institute.

### GEO

[Gene Expression Omnibus (GEO)](https://www.ncbi.nlm.nih.gov/geo/) is a database for gene expression profiling and RNA methylation profiling managed by the National Center for Biotechnology Information (NCBI). Array- and sequence-based data are accepted.

### gnomAD

[gnomAD (the Genome Aggregation Database)](https://gnomad.broadinstitute.org/), originally launched in 2014 as the Exome Aggregation Consortium (ExAC), is a **resource** developed by an international coalition of investigators [**organization**], with the goal of aggregating and harmonizing both exome and genome sequencing data from a wide variety of large-scale sequencing projects, and making summary data available for the wider scientific community. Broad Institute contributes data storage, computing resources, and human effort.

### VEP

The [Variant Effect Predictor (VEP)](https://useast.ensembl.org/info/docs/tools/vep/index.html) is part of Ensembl and Ensembl Genomes and allows the user to explore and analyze the effect that the variants (SNPs, CNVs, indels or structural variations) have on a particular gene, sequence, protein, transcript or transcription factor.

# Projects & Programs

### 1000 Genomes

- launched in January 2008; completed in 2015
- 2,504 samples
- Demographics difficult to summarize
- Data provided through [IGSR](https://www.internationalgenome.org/data)

### All of Us

The [All of Us](https://allofus.nih.gov/) Research Program is a historic effort to collect and study data from one million or more people living in the United States.

Demographics (n=409,420):

| Population |       n |     % |
|------------|--------:|------:|
| white      | 222,660 | 54.38 |
| black      |  77,080 | 18.83 |
| hispanic   |  64,680 |  15.8 |
| >1         |  16,280 |  3.98 |
| asian      |  13,840 |  3.38 |
| other      |   7,180 |  1.75 |
| skip       |   5,220 |  1.27 |
| decline    |   2,560 |  0.63 |
| total      | 409,500 |       |

[source](https://databrowser.researchallofus.org/survey/the-basics/race)

245,400 WGS samples

### CCDG

The [Centers for Common Disease Genomics](https://ccdg.rutgers.edu/) (CCDG) are a collaborative large-scale genome sequencing effort comprehensively identifying rare risk and protective variants that contribute to multiple common disease phenotypes.

### gnomAD

Demographics

| Population               | overall |
|--------------------------|--------:|
| African/African American |  20,744 |
| Amish                    |     456 |
| Latino/Admixed American  |   7,647 |
| Ashkenazi Jewish         |   1,736 |
| East Asian               |   2,604 |
| European (Finnish)       |   5,316 |
| Middle Eastern           |     158 |
| European (non-Finnish)   |  34,029 |
| South Asian              |   2,419 |
| Other                    |   1,047 |
| XX                       |  38,947 |
| XY                       |  37,209 |
| Total                    |  76,156 |

### TOPMed

The Trans-Omics for Precision Medicine (TOPMed) program, sponsored by the National Institutes of Health (NIH) National Heart, Lung and Blood Institute (NHLBI), is part of a broader Precision Medicine Initiative, which aims to provide disease treatments tailored to an individual’s unique genes and environment. TOPMed data are being made available to the scientific community as a series of "data freezes": genotypes and phenotypes via dbGaP; read alignments via the Sequence Read Archive (SRA); and variant summary information via the Bravo variant server and dbSNP.

| population             |      n |  % |
|------------------------|-------:|---:|
| European               | 72,300 | 40 |
| African                | 51,110 | 29 |
| Hispanic               | 34,060 | 19 |
| Asian                  | 14,700 |  8 |
| Other/Multiple/Unknown |  7,240 |  4 |

[source](https://topmed.nhlbi.nih.gov/#Participant%20Diversity)

### Summary

|               |       n |      European |      African |     Hispanic |       Asian |          NA |
|---------------|--------:|--------------:|-------------:|-------------:|------------:|------------:|
| 1000 Genomes  |   2,504 |               |              |              |             |             |
| All of Us     | 409,420 | 222,660 (54%) | 77,080 (19%) | 64,680 (16%) | 13,840 (3%) | 31,240 (8%) |
| Genebass/UKBB | 394,841 | 367,300 (93%) |   6,700 (2%) |       0 (0%) |  5,200 (1%) | 15,800 (4%) |
| gnomAD        |  76,156 |  39,345 (52%) | 20,744 (27%) |  7,647 (10%) |  5,023 (7%) |  3,397 (4%) |
| MVP           | 104,923 |  72,939 (70%) | 24,623 (23%) |   5,554 (5%) |    687 (1%) |  1,120 (1%) |
| TOPMed        | 179,410 |  72,300 (40%) | 51,110 (29%) | 34,060 (19%) | 14,700 (8%) |  7,240 (4%) |

- Note: NA = Other, multiple, declined, skipped, etc.
- Note: All of Us claimed 245,400 WGS samples (from a population of 409,420)
- Note: UKBB ancestry statistics from [NPR article](https://www.npr.org/sections/health-shots/2019/08/22/752890414/lack-of-diversity-in-genetic-databases-hampers-research)
- 430,000 “white British ancestry
- 78,296 UKBB “non-white British

# Biobanks

UK Biobank - ~500k volunteers - longitudinal study over period of 30 years -
connects genotypes & phenotypes - founded by Sir Rory Collins - overseen by a
board chaired by Lord Kakkar that is accountable to the Medical Research
Council & Wellcome Trust (the constituent organizations of the company) -
funded by the UK Department of Health, the Medical Research Council, the
Scottish Executive, and the Wellcome Trust medical research charity.

All of Us - Mayo Clinic in Rochester, Minnesota

# Organizations

- Genome Reference Consortium (GRC)
- 
- (under NIH)
-  (under NIH)
- Centers for Common Disease Genomics (CCDG) (under NHGRI)

# Summary of organizations and resources
```
European Molecular Biology Laboratory (EMBL)
└── European Bioinformatics Institute (EBI)
    └── Ensembl
        ├── Annotations
        └── Variant effect predictor (VEP)
        
United Kingdom Research and Innovation (UKRI)
└── Medical Research Council (MRC)
    └── UK Biobank
        └── Genebass
        
Department of Health and Human Services (HHS)
├── National Institutes of Health (NIH)
│   └── National Human Genome Research Institute (NHGRI)
│       ├── Centers for Common Disease Genomics (CCDG)
│       └── Human Genome Project (HGP)
├── National Heart, Lung and Blood Institute (NHLBI)
│   └── Trans-Omics for Precision Medicine (TOPMed) Consortium
│       ├── Data on dbGaP (database of Genotypes and Phenotypes)
│       └── Bravo variant browser
└── National Library of Medicine (NLM)
    └── National Center for Biotechnology Information (NCBI)
        ├── GenBank sequence database
        ├── Single Nucleotide Polymorphism Database (dbSNP)
        ├── Online Mendelian Inheritance in Man (OMIM)
        └── PubMed
        
University of Washington
└── Combined Annotation Dependent Depletion (CADD)

Broad Institute
├── Genome Aggregation Database (gnomaD)
└── Hail
```
