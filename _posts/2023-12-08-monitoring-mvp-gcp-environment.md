---
layout: post
title:  "#38 Monitoring the MVP environment on Google Cloud"
date:   2023-10-13 10:05:00 -0800
author: Paul Billing-Ross
categories: jekyll update
faq:
- question: "How are Neo4j database credentials stored?"
  answer: "Using Google [Secret Manager](https://cloud.google.com/secret-manager)."
- question: "Where is the Trellis configuration file?"
  answer: "The `trellis-config.yaml` file is at the base of the Trellis bucket."
- question: "How can I interact with the Neo4j database?"
  answer: "Using the Neo4j Desktop client on your local computer."
- question: "How can I monitor behaviour of Trellis systems?"
  answer: "Using the Monitoring dashboard."
---

## Monitoring Trellis

Trellis is a an event-driven system that tracks data and orchestrates workflows using metadata. This metadata is stored in a Neo4j graph database and is used to activate database triggers. The triggers will launch compute jobs to process new data as soon as it is available. The (2) easiest ways to monitor Trellis are by a) connecting to the Neo4j database and interacting with it through the [Neo4j Browser](https://neo4j.com/docs/browser-manual/current/) that comes with the [Neo4j desktop](https://neo4j.com/developer/neo4j-desktop/) client that you can download [here](http://neo4j.com/download) and b) by checking the Monitoring dashboard or by creating additional dashboards/widgets.

### Neo4j Browser
You should find the database credentials of each Neo4j instance in the Secret Manager of the same project the database is running in under the name "trellis-neo4j-credentials". Trellis behavior is governed by a configuration file call "trellis-config.yaml" in the Trellis bucket of each project it is deployed in. Follow the steps in the activity to try interacting with the database via browser (Stanford staff only).

Also, Trellis behavior is governed by a configuration file called "trellis-config.yaml" in the Trellis bucket of each project it is deployed in. Try reading the object without taking it out of the cloud environment (because you should never take anything out of the cloud environment).

#### Activity
- Download the Neo4j Desktop Client
- Retrieve the Neo4j database credentials
- Connect to the database from your local Neo4j client
- Try running some queries to determine the state of MVP data (see below) 
- **Bonus**: Try creating a Cypher query to find any samples that were delivered in the last week, but haven't finished variant calling. Use the browser to navigate the graph and identify the last completed step. *(Hint: don't start from scratch)*

### Trellis Monitoring Dashboard
The other way to keep an eye on Trellis and the systems that run it (compute instances, cloud functions, etc.) is using [Monitoring Dashboards](https://cloud.google.com/monitoring/dashboards/api-examples). I've created one spefically [for Trellis](https://console.cloud.google.com/monitoring/dashboards/builder/dc56b2f7-f69b-4527-b054-3f7bcc1852a7;duration=P1D?authuser=0&project=gbsc-gcp-project-mvp). If you are not Stanford Staff, you can check out the slide decks below to get a better idea of what the dashboard looks like.

#### Activity
Using the Monitoring dashboard, answer these questions:
- How long has the average Neo4j query taken to run in the last 7 days?
- How long has the longest query taken?
- How much MVP data are we storing?
- Which storage buckets host the most data?

#### From [the Archives](https://drive.google.com/drive/folders/1acIugd7SU6XgWR3vTyqw887_DKzkVFxG)
Because we are not currently running variant calling it's harder to get a feel for what the dashboard *should* look like, but you can look back at some of my old slide decks to see get an idea what normal and aberrant patterns look like.

- August 5, 2020: [Troubleshooting Neo4j query timeouts](https://docs.google.com/presentation/d/1ENFS4pJQ4U-fk1g4zCnYbLTzkR_sKb1OJVpJFvtuCPQ/edit#slide=id.p)
- May 27, 202: [Running out of genomes](https://docs.google.com/presentation/d/1DutQ5kylQloA6rmdwMHUhFit0d0k8QEzHJL7sjav04c/edit#slide=id.p)
- May 12, 2020: [Trellis v1 performance](https://docs.google.com/presentation/d/193S-oS4EaiIA5bRAZsZLsv0Kjt6dHf8KDqK-LkWM5hY/edit#slide=id.p)

## Core Trellis repositories
- [https://github.com/StanfordBioinformatics/trellis-mvp-functions](https://github.com/StanfordBioinformatics/trellis-mvp-functions): Trellis is an application built using [Cloud Functions](https://cloud.google.com/functions?hl=en). This repository contains the source for these functions as well as the `cloudbuild.yaml` configuration that [Cloud Build](https://cloud.google.com/build?hl=en) uses to automatically deploy function updates. 
- [https://github.com/StanfordBioinformatics/trellis-mvp-terraform](https://github.com/StanfordBioinformatics/trellis-mvp-terraform): Trellis relies on numerous Google Cloud services and features. Configuration of all these environment variables is performed programmatically by coding them into Infrastructure as Code configuration files. I have used Terraform to manage MVP cloud infrastructure as code. Note that right now deployment is handled two ways: via Terraform for the cloud environment and via Cloud Build for live function updates. I've done this because it's mostly the functions that need to be updated and I've found it to be much simpler to deploy those updates automatically through GitHub than by manually running Terraform which has a tendency to over redeploy resources that do not need to be updated because some minor configuration detail has changed on the platform side.
- [https://github.com/StanfordBioinformatics/trellis-docs](https://github.com/StanfordBioinformatics/trellis-docs): Source for Trellis [ReadTheDocs](https://trellis-data-management.readthedocs.io/en/latest/).
- [https://github.com/va-big-data-genomics/trellis-mvp-data-modelling](https://github.com/va-big-data-genomics/trellis-mvp-data-modelling): Scripts I have used to update the Trellis metadata model (e.g. adding new types of data).
- [https://github.com/va-big-data-genomics/trellis-mvp-gatk](https://github.com/va-big-data-genomics/trellis-mvp-gatk): Source for the GATK workflow we use for variant calling. 

### Activity
Figure out which branch of the `trellis-mvp-gatk` repository is being deployed to production. Hint: check the Cloud Build configuration.

## Cypher queries for checking the state of MVP data

### 1. How many sequencing samples have been delivered?

```
MATCH (n:PersonalisSequencing)
RETURN COUNT(DISTINCT n)
```

### 2. How many samples have variants called?

```
MATCH (n:PersonalisSequencing)<-[:WAS_USED_BY]-(:Sample)<-[:GENERATED]-(:Person)-[:HAS_BIOLOGICAL_OME]->(:Genome)-[:HAS_VARIANT_CALLS]->(v:Merged:Vcf)
RETURN COUNT(DISTINCT n)
```

### 3. How many samples have been delivered within the last 48 hours?

```
MATCH (n:PersonalisSequencing)
WHERE n.timeCreatedEpoch > (datetime().epochSeconds - 172800)
RETURN COUNT(n)
```

### 4. How many samples have started variant calling in the last 7 days?

```
MATCH (n:PersonalisSequencing)-[:GENERATED]->(:Fastq)-[:WAS_USED_BY]->(:Job:FastqToUbam)
WHERE n.timeCreatedEpoch > (datetime().epochSeconds - 604800)
RETURN COUNT(DISTINCT n)
```

### 5. How many samples have completed variant calling in the last 7 days? 

```
WITH datetime().epochSeconds - 604800 AS sevenDayEpoch
MATCH (n:PersonalisSequencing)<-[]-(:Sample)<-[]-(:Person)-[:HAS_BIOLOGICAL_OME]->(:Genome)-[:HAS_VARIANT_CALLS]->(v:Merged:Vcf)
WHERE v.timeCreatedEpoch > sevenDayEpoch
RETURN COUNT(DISTINCT n)
```

### 6. How many Fastqs are not connected to a PersonalisSequencing node? (Error: broken relationship)

```
MATCH (n:Fastq)
WHERE NOT (:PersonalisSequencing)-[]->(n)
RETURN COUNT(n)
```

### 7. How many samples have finished variant calling and have all essential QC documents?

```
MATCH (n:PersonalisSequencing)<-[:WAS_USED_BY]-(:Sample)<-[:GENERATED]-(:Person)-[:HAS_BIOLOGICAL_OME]->(g:Genome)-[:HAS_VARIANT_CALLS]->(v:Merged:Vcf)
WHERE (g)-[:HAS_QC_DATA]->(:Flagstat)
AND (g)-[:HAS_QC_DATA]->(:Fastqc)
AND (g)-[:HAS_QC_DATA]->(:Vcfstats)
RETURN COUNT(g)
```

### 8. How many QC documents does every sample have?

```
MATCH (n:PersonalisSequencing)<-[:WAS_USED_BY]-(:Sample)<-[:GENERATED]-(:Person)-[:HAS_BIOLOGICAL_OME]->(g:Genome)-[:HAS_VARIANT_CALLS]->(v:Merged:Vcf)
WITH g
MATCH (g)-[:HAS_QC_DATA]->(qc)
WITH g, COLLECT(qc) AS qcs
RETURN COUNT(g), SIZE(qcs)
```

## Discuss
Join the discussion of this post on [GitHub](https://github.com/orgs/va-big-data-genomics/discussions/37)!
