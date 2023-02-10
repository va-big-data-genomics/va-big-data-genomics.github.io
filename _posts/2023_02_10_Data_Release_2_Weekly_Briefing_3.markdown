---
layout: post
title:  "#9 Data Release 2 Weekly Briefing 3"
date:   2023-02-10 10:11:02 -0800
author: Joe Sarro 
categories: jekyll update
---
## Current updates 

I have been investigating the issue involving high missingness rates on the X chromosome. I have succeeded in creating a plot that will show missingness per variant for individual chromosomes. Alternative methods outside of Hail may be needed to generate additional plots that show missingness for all chromosomes, as shown in Bryan’s [box and whisker plot]( https://github.com/va-big-data-genomics/mvp-wgs-snp-indel-release/blob/main/SNPs-Indels/data_release_2023/WGS_Release2_Missingness_By_Chromosome.pdf), due to the limited plotting tools build in Hail.

Using this methodology, I was able to track missingness on the X chromosome while updating our methods to filter genotypes. I was also able to look at the affect of using Gnomads filtering methods on a subset of our data. After looking at our filter criteria, it appears that high rates of missingness on the X chromosome may be due to low genotype quality scores. 

More detailed analysis can be viewed in the [missingness analysis notebook](https://drive.google.com/file/d/157GIt0LxN9LOdbKmCg4mxhHWAIXoytOk/view?usp=share_link) that I generated. Unfortunately, Bokeh plots do not save when closing the notebook. Because of this, the previous link is to an html version that you can hopefully view in your web browser. 

Some questions to consider are whether or not we want to instill sex specific filters for the X chromosome and how we want to deal with the issue of genotype quality scores potentially being lower on the X chromosome? For the former, I have been looking into implementing the Hail Boolean expression [impute_sex()](https://hail.is/docs/0.2/methods/genetics.html#hail.methods.impute_sex).


