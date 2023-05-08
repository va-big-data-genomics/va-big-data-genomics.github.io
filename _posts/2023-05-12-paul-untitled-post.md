---
layout: post
title:  "# (n) pbilling untitled"
date:   2023-05-8 10:10:00 -0800
author: Paul Billing-Ross 
categories: jekyll update
tags: [Mermaid]
mermaid: true
---

# Modelling studies in Neo4j

A primary motivation for us to model workflow data as graphs was to capture the complete data lineage or provenance of data we generated for the sake of auditing the validity of data and reproducing analyses. However, the value of tracking data lineage can also be used to inform cohort selection and downstream analyses. A major factor in choosing samples for analyses is considering which ones they have already been used in and graphs represent an ideal model for this use case. Here I will show how we model studies in Neo4j and how that model can be used to find overlapping samples between studies.

## Importing study participants into Neo4j

- Use a standard Python script to handle import
- Configure the script using a YAML file
- Reference secrets as environment variables in the YAML file
- Store the secrets in a Secret Manager secret

Reference: https://dev.to/mkaranasou/python-yaml-configuration-with-environment-variables-parsing-2ha6

### New model architecture

<div class="mermaid">
graph LR
    A --- B
    B-->C[Happy]
    B-->D(Sad);
</div>

<div class="mermaid">
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
</div>

<div class="mermaid">
    sequenceDiagram
    parse[parse-study-participants] ->> dbquery[db-query]: QueryRequest
    dbquery ->> dbtriggers[db-triggers]: QueryResponse
    dbtriggers -) dbquery: QueryRequest
</div>


### Order of database queries

- Create study node
- Create participant nodes
- Relate participant nodes to study node & update study node with participant count

# Philosophy for releasing data

Our primary objective is to deliver whole genome sequencing data to Million Veteran Program researchers. When considering how to deliver data, the context is just as important as the content. A few questions for us to consider regarding delivery:

- Where should data be made available?
- In what format(s) should it be available?
- How should data be filtered for quality before delivery?
- Should data be filtered for quality?
- What kinds of questions are researchers trying to answer with this data?
- How often should we deliver data?
- What kinds of data should we deliver?
- Should different kinds of data be delivered separately or together?

## Style guide: study naming conventions

Do not name things based on the number of elements. A core tenet of "big data" is scalability and a resource that is named according to how many samples it is designed for is inherently **not** scalable. Every production resource you create should be designed to scale down to 1 and scale up to 1 million. If not, then you are just going to have to remake it every time we want to add more samples, which is always. You are not building equity within the group, you are just spinning on a bioinformatics hamster wheel. Don't be a hamster.

Instead, I recommend naming things based on iterations (e.g. "wgs-data-release-1", "telomere-pilot", "telomere-study-2"), starting from (1), where a "pilot" indicates that the purpose of the study is research and development rather than releasing data to MVP researchers.

# Discuss
Join the discussion on our <ins>[GitHub](https://github.com/orgs/va-big-data-genomics/discussions/18)</ins>!
