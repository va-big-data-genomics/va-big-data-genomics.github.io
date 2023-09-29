---
layout: post
title:  "#34 Thoughts on the NHLBI Big Data Integration workshop"
date:   2023-09-29 13:00:00 -0700
author: Paul Billing-Ross
categories: jekyll update
---

## All of Us
- OMOP seems to be the standard for representing clinical data; if we are going to design systems to interact with clinical data, we should use OMOP; something we have already discussed [link to graph OMOP model].
- A good place to start with MVP GWAS/PheWAS would be replicating studies published by other groups. With PheWAS, AoU was able to replicate 38/38 ClinGen gene-cancer associations and 164/235 OMIM gene-phenotype associations.
- With whole genome sequencing of 250,000 individuals, AoU observed 1 billion variants with 600 million not observed in gnomAD. We see 600 million with variants not in gnomAD. We had been trying to compare our variants against dbSNP but maybe comparing against gnomAD would be easier or more appropriate? Curious why they chose gnomAD.
- In performing a large-scale GWAS, All of Us researchers found robust correlations between polygenic risk scores blood pressure and hypertension in African and European ancestry populations (Keaton, Kamali et al., in revision). Their study included 84 thousand individuals of African and over 1 million individuals of European ancestry. I'm guessing based on the numbers that this was genotyping array data? We have over 30 thousand African ancestry individuals in our dataset; trying to replicate this result could provide an interesting proof-of-concept of applying polygenic risk scores to MVP whole genome sequencing data.

## Million Veteran Program
- We have the largest cohort of people with African American ancestry (~150 thousand). We should emphasize this when we share data by highlighting the African American population in our validation studies.
- The Million Veteran Program is also collecting methylation, metabolomics, and proteomics data. I don't know of any specifics of how that data will be uses. But, as we plan future features, validation studies, and additional technical infrastructure, it's worth keeping an eye out for methods that could enable multiomic studies that integrate these other data types.
- Saiju showed scatterplots comparing alternate allele frequencies in the Million Veteran Program population with frequencies in 1000 Genomes populations. I think his point was that, other than the European populations, there is very poor correlation of within population allele frequencies between MVP and 1000 Genomes and so it's worth continuing to sequence and study these populations. But also, his group randomly chose 50,000 genetic markers for comparison. I would be interested to see the correlation between common variant alternate allele frequencies in our dataset and other large cohort datasets. I think that could be another interesting point of validation.

## Notes on sharing research

### We need better ways to cite and share papers
For the sake of citing a publication in their slide, someone included the following notation at the bottom of their slide: "Wang et al. (2023) J Hum Genetics". Punching this into Google does not return the relevant result. Similarly, putting it into Google Scholar does not either. When I copied it into PubMed it gave me a warning that "Your search was processed without automatic term mapping because it retrieved zero results." Even the All of Us publication browser only provides search functionality based on the title of the paper. 

I finally found the paper by typing the slide title "Common and Rare Variants Associated with Cardiometabolic Traits in All of Us Participants" into Google, after which I found the paper, full title: "Common and rare variants associated with cardiometabolic traits across 98,622 whole-genome sequences in the All of Us research program". I can understand why they didn't try to include the entire title in the citation: it's too long. So, what's a better alternative?

Well, what are the most obvious identifying features of this paper? We've got a few in the citation: author, publication year, and journal. The problem is that these are all too broad. There are lots of authors named "Wang" and lots of papers have been published in the Journal of Human Genetics. The journal name is also too long. They abbreviated it for the citation. That leads to more ambiguity because it can be abbreviated multiple ways: J Hum Genetics, J Human Genet., J Human Genetics, etc. It's a data harmonization nightmare. What are better features for search?

For starters the PubMed or PubMed Central IDs. Why they use two different IDs? I don't know, but they are short and universally unique IDs. Punching the PMID for this paper directly into Google got me the PubMed link to the paper with the first hit. Weirdly, the PMCID didn't pull any relevant results. So, I would say recommend PubMed ID for citing academic journal publications in presentations. There is a similar ID system used by bioRxiv that could be used for sharing preprints.

But also, I don't think we need to be locked into one ID system for finding publications. All of Us maintains their own list of [publications](https://www.researchallofus.org/publications/). They could also assign their own IDs (e.g. AoU22). I think the most important thing is that search engines can recognize these identifiers and return the appropriate results. I could imagine including structured data definitions on the publication list, so that search engines could add the ID mappings to their knowledge bases.

### How to track methods used to analyze large shared datasets?
In reference to all the studies being performed with All of Us data, someone asked whether investigators could be compelled to share their code publicly or on the [Terra](https://terra.bio/) platform. Denny's response was basically "no", and I think he expressed hope that publications would encourage investigators to publish their code. I think there's a few flaws with this model: 

1) It requires researchers to publish research through mainstream scientific publications. This is a non-starter for me given how terrible and exploitative the academic publishing industry is. We should be moving away from publications being controlled by private companies. 
2) Only code that generates results those private companies think is worth publishing will be shared. What about code that is used for reproducing existing results or that generates negative results? This code could be just as valuable as code that generates novel results.
3) It assumes that researchers will self-publish complete or valid code. They will not. Have you ever looked at a GitHub repo published as a companion to a paper? It has one README that contains at most 300 words and a bunch Perl/Python/R scripts with zero comments or documentation. You can't rely on users to manually publish accurate methods, there is too much human error involved.

So, what's the answer? Automation. If researchers are doing on their computation on a single platform, you could *theoretically* capture the complete provenance of every data object they create. But the more freedom you give them, the harder that will be.

Our Stanford MVP environment can solve that problem by providing a curated API that provides a single point of interface between our data and the researcher. Because all of their operations are mediated through our system, we can document every workflow they use to generate results and share them publicly. The cost of this is that users will be limited in how they can interact with the data. And once they take the results from our environment and manipulate them in their own environment, we can no longer track their process. But I still think we could track the high level workflows used to generate the data that someone else would need to reproduce or replicate their analysis. For instance, we could track of the association tests (e.g. GWAS) run in our system and share those results/methods with other researchers.

## Discuss
Join the discussion of this post on [GitHub](https://github.com/orgs/va-big-data-genomics/discussions/37)!