---
layout: post
title:  "Data Release 2 Weekly Briefing"
date:   2023-01-27 06:10:02 -0800
author: Joe Sarro
categories: jekyll update
---

I am currently generating a list of tasks, implemented as issues, that can be found on our git repository. Each task will be populated with a detailed description, a team member assigned, and any perquisites needed to complete the task. A workflow of these tasks is described in Fig. 1. 

![Fig 1. Graph describing the workflow of tasks needed to be performed to complete Data Release 2](/assets/Data-release-2-task-workflow-with-labels.png)

Three additional issues recently created are in relation to discussions we had in our last, [(01/20/2023)](https://github.com/va-big-data-genomics/va-big-data-genomics.github.io/blob/main/_posts/2023-01-20-Data-Release2-Update-Joe-Sarro.markdown), group meeting. First, the question of whether mitochondrial DNA is included in our data came up. [The answer to this question is no](https://github.com/va-big-data-genomics/mvp-wgs-snp-indel-release/issues/39). It was also mentioned that the Boston team had used a minor allele frequency filtering cutoff of 5% to run kinship analysis. This is [something our pipeline is lacking](https://github.com/va-big-data-genomics/mvp-wgs-snp-indel-release/issues/38) and may be worth adding for our own kinship analysis. 

Lastly, a review of our genotype filtering may need to be conducted. [We are currently filtering pseudoautosomal regions on the sex chromosomes as autosomes](https://github.com/va-big-data-genomics/mvp-wgs-snp-indel-release/issues/37). Depending on how genome alignment and variant calling are conducted, this may need to be changed.
