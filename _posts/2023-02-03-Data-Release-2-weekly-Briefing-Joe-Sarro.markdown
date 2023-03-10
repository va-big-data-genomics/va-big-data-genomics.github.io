---
layout: post
title:  "#6 Data Release 2 Weekly Briefing 2"
date:   2023-02-03 10:10:02 -0800
author: Joe Sarro 
categories: jekyll update
---
# Kinship

We have received updated kinship results from the Boston team. I have written a notebook to process this table and generate a list of first- and second-degree relationships (currently in a pull request). It was noticed during processing that some kinship comparisons were not grouping into a degree relationship. Looking at the [king manual](https://www.kingrelatedness.com/manual.shtml), the coefficient ranges for first and second degree relationships are ambiguous. The manual includes coefficients between [0.177, 0.354]as first degree, [0.0884, 0.177] as second degree, and [0.0442, 0.0884] as third degree. This creates an overlap for 0.177 in first- and second-degree groupings and 0.0884 in second- and third-degree relationships. Our Rel1 methodology used the following cutoffs:

first degree | > 0.177 | <br>
second degree | <=0.177  | >0.0884 <br>

It should be noted that twin samples [ >0.354] are grouped with first degree samples in our dataset.

By changing the cutoffs to:

first degree | >=0.177 | <br>
second degree | <0.177 | >= 0.0884 <br>

All Boston data was able to be grouped.

# X and Y Chromosomes

As discussed in past meetings, Bryan had discovered increased missingness on the X chromosome and a high number of variants on the Y chromosome in our current dataset. I have compared the ratio of the number of variants on the Y chromosome to chromosome 1 and chromosome 22 (Currently in a notebook under a pull request). Chromosome 1 was chosen based on the fact it is the chromosome with the highest number of variants, and chromosome 22 was chosen due to it being the autosome with the least number of variants. In Data Release 1 the Y chromosome contained roughly 4% of the number of variants found in chromosome 1. This increased to 7% in Data Release 2. In Data Release 1 the Y chromosome contained roughly 23% of the variants in chromosome 22. This increased to 41% in Data Release 2. An increase in total male samples may explain this increase in variants observed on the Y chromosome. The question of whether this increase is significant enough to be explored needs to be discussed. 

As noted in [git issue 42](https://github.com/va-big-data-genomics/mvp-wgs-snp-indel-release/issues/42), we discovered missingness data generated by Jina Song for Data Release 1. This Data shows similar missingness rates for the X chromosome in Data Release 1 that we are seeing for the current release. We are now left with questions about whether these rates should have been accepted for Data Release 1 and if we should be using stricter criteria for the current release.

# Release Progress

We are currently tracking Data Release 2 progress through [git](https://github.com/orgs/va-big-data-genomics/projects/4/views/1). A number of the assigned tasks require a final matrix table to proceed. We are hoping to generate this table by the end of next week, as I will be going on leave after that.

**Discuss** this update on our [GitHub](https://github.com/orgs/va-big-data-genomics/discussions/4)!