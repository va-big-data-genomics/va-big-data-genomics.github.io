---
layout: post
title:  "#19 Adding study participants to Neo4j"
date:   2023-05-12 10:10:00 -0800
author: Paul Billing-Ross 
categories: jekyll update
tags: [Mermaid]
mermaid: true
---

# Modelling studies in Neo4j

A primary motivation for us to model workflow data as graphs is to capture the complete data lineage or provenance of data we generate for the sake of auditing the validity of data and reproducing analyses. However, the value of tracking data lineage can also be used to inform cohort selection and downstream analyses. A major factor in choosing samples for analyses is considering which ones they have already been used in and graphs represent an ideal model for this use case. Here I will describe how I am updating Trellis to  model studies in the Neo4j metadata store and how that model can be used to find overlapping samples between studies.

## Updated Trellis database model

In the updated database model, the person who provided the biological sample is represented by three different nodes: `Person`, `Sample`, and `Participant`. The `Person` node represents the root node of all the data generated by sample(s) provided by the individual. This node could contain basic information about this individual such as age, gender, and ethnicity. The `Sample` node(s) refer to individual sequenced samples generated from the person. If someone provided a blood sample that was subsampled to be sequenced twice, each of those sequencing results should have separate shipping IDs and separate `Sample` nodes. Because we only handle the electronic data generated from sequencing these samples, we don't currently attach metadata to these nodes. The `Participant` nodes store study specific metadata related to the individual such as biological phenotypes (e.g. telomere length). Because the phenotype classification can depend on the study methods, we create separate `Participant` nodes for every study an person's samples are used for. Because a person can have multiple samples, we use the `PROVIDED_SAMPLE` relationship to indicate which was used in a study.

<div class="mermaid">
    graph TD
        study((Study)) -- HAS_PARTICIPANT --> par((Participant))
        par -- IS --> person((Person))
        person -- GENERATED --> sample((Sample))
        par -- PROVIDED_SAMPLE --> sample
</div>
Fig 1. Diagram describing the new graph model for connecting people to studies in the Trellis metadata store.

## Updated Trellis architecture

A cloud function named `add-study-participants` will listen for new objects added to a `trellis-studies` bucket. When a new object named "*-trellis-study.yaml" is added, it will read the study configuration file in order to add the study information to the Trellis Neo4j metadata store. When users want to add a study to the database they will upload the YAML file as well as a CSV file with the relevant metadata for each study participant. This will include a "sample" column which will be used to map the `(:Participant)` nodes to their corresponding `(:Sample)` and `(:Person)` nodes.

<div class="mermaid">
    graph TD
        gcs(trellis-studies) -- CloudEvent --> add-study-participants
        add-study-participants -- QueryRequest --> db-query
        db-query -- QueryResponse --> db-triggers
        db-triggers -- QueryRequest --> db-query
        db-query -- Cypher --> neo4j((Neo4j DB))
        neo4j --> neo4j.Result --> db-query
</div>
Fig 2. Architecture diagram of Trellis resources involved in adding study information to the Trellis metadata store.


**Example study configuration object**

```
--- !Study
required_fields:
    name: WgsTelseqPilot
    type: Pilot
    lead: "Prathima Vembu"
--- !Participants
csv_path: telseq_results.csv
required_fields:
    sample: 0
```

I want to be able to populate the `Study` and `Participant` nodes with metadata fields that are custom to each study, while still using [parameterized queries](https://trellis-data-management.readthedocs.io/en/latest/v1-3-update.html#switching-to-the-neo4j-database-driver), so instead of using a predefined query, the `add-study-participants` function will dynamically generate a parameterized query similar to as is done by the `create-blob-node` function. All queries will be processed by the Trellis `db-query` function.

## Steps to adding study participants to Neo4j

### Create the Study node

First, the `add-study-participants` function will create the study node with relevant metadata.

**Example Cypher query to create Study node**
```
MERGE (study:Study {name: $name})
SET study.notes = $notes, 
    study.type = $type, 
    study.lead = $lead 
RETURN study
```

### Create the Participant nodes
Then, it will send a set of queries to create the participant nodes. If I am understanding the <ins>[Pub/Sub documentation](https://cloud.google.com/pubsub/quotas)</ins> correctly, the maximum size of a Pub/Sub message is 10MB. Because we want this function to be scalable, the function will split the data from the CSV into multiple query request messages if it exceeds a predefined threshold (e.g. 9MB).

**Example query to add study participants to database**

```
UNWIND $data AS participant_entry
MERGE (participant:Participant {
        study: $studyName,
        sample: participant_entry.sample
})
SET participant.telomereLengthEstimate = toFloat(participant_entry.lengthEstimate
RETURN len(participant)
```

### Relate Participant relationships
The result of the participants query will trigger (2) sets of query requests. The first to connect the `Participant` nodes to the `Study` and the second to connect `Participant` nodes to their corresponding `Sample` and `Person` nodes, using the "sample" field. 

<div class="mermaid">
    sequenceDiagram
    add-study-participants ->> db-query: QueryRequest: createStudyNode
    db-query ->> db-triggers: QueryResponse: createStudyNode
    add-study-participants ->> db-query: QueryRequest: createParticipantNodes
    db-query ->> db-triggers: QueryResponse: createParticipantNodes
    db-triggers ->> db-query: QueryRequest: relateParticipantsToStudy
    db-query ->> db-triggers: QueryResponse: relateParticipantsToStudy
    db-triggers ->> db-query: QueryRequest: relateParticipantsToSamples
    db-query ->> db-triggers: QueryResponse: relateParticipantsToSamples
    db-triggers ->> db-query: QueryRequest: addStudyParticipantCount
</div>
Fig 3. Sequence diagram describing how study participants will be added to the Trellis metadata store.

### Count study participants
Finally, the result of relating participants to sample nodes will trigger a query that will update the Study node with a count of all the study participants.

## Implementation roadmap
- Create a GitHub project
- Create an `add-study-participants` Trellis function
- Write queries
  - createStudyNode
  - createParticipantNodes
  - relateParticipantsToSamples
  - addStudyParticipantCount
- Write methods to dynamically generate queries
- Add classes to trellisdata so can read from YAML
  - Study
  - Participant

## Relevant Trellis design principles
- **As much as possible, use existing architecture**. In this case, I've only added one new function, `add-study-participants`. The other functions involved are existing core Trellis functions.
- **Change configuration files, not code.**
- **Each database query should do one thing.** This philosophy applies to queries as well as serverless functions. By narrowing the scope of each query/function to a single operation, we can reuse them to suit a broad range of future use cases and also reduce the risk of them breaking as the system continues to grow and change.

### UPDATE: How to handle batch queries?
A design question that I encountered when thinking about adding study participants was how to handle queries that perform batch processing and generate a lot of results. Trellis is generally designed to handle entitities individually. For instance, when a `Fastq` is added to cloud storage a single query runs to add it as a node to the database and then a separate query is triggered to relate to to a `Sample`. In this case, I am running one query to add many nodes and then I subsequently need to create relationships for each of those nodes.

Following my existing design patterns, I could return every new node (n) as a separate result and trigger a separate relationship query for each of them. But if (n=1,000,000,000) that will generate (n) queries and overload the database. It's also unnecessary, because I only need to run a single query to create all the relationships.

```
MATCH (study:Study), (participant:Participant)
WHERE study.name = $study
AND participant.study = $study
MERGE (study)-[:HAS_PARTICIPANT]->(participant)
```

Additionally, to improve query performace of large batch queries, I can use the `apoc.periodic.iterate()` function from the Neo4j "Awesome Procedures on Cypher" (APOC) library.

```
CALL apoc.periodic.iterate(
    "MATCH (study:Study), (participant:Participant)
    WHERE participant.study = $study
    AND study.name = $study RETURN participant, study",
    "MERGE (study)-[has:HAS_PARTICIPANT]->(participant)",
    {batchSize: 200, parallel: true, params:{study: $study}}
)
```

However, a limitation of this method is that I can't return graph objects, only summary metrics describing changes to the database. Trellis database triggers are only designed to respond to graph objects (nodes or relationships), so again, this doesn't fit with the standard Trellis paradigm of modelling database operations. So, I may have to go back to the drawing board on how these operations will be linked together. If this is a distinctly different use case from standard Trellis operations it may be better to implement a different workflow than try to fit a round peg in a square hole.

# Overlap between studies
The process for automating the process of adding studies to Trellis is still ongoing, but in the interest of getting results, I've created some [scripts](https://github.com/va-big-data-genomics/trellis-mvp-data-modelling/tree/main/database-update-scripts) to add the Telseq Pilot and Whole Genome Sequencing Data Release 2 studies to the database.  And because all of the relationships between people and studies are tracked in the same graph database, we can use graph traversal to find samples that belong to multiple studies.

**Query using graph traversal to find overlapping sample**
```
MATCH path=(s:Study {name:"WgsTelseqPilot"})-[:HAS_PARTICIPANT]->(p)-[:IS]->(person:Person)<-[:IS]-(:Participant)<-[:HAS_PARTICIPANT]-(:Study {name:"WgsDataRelease1"})
RETURN COUNT(path)
```

## WGS Data releases

We see 99.9% overlap between Data Release 1 & 2. This difference could be the result of a technical issue with the database or a real difference in samples between releases.

| Description | Sample count |
| ----------- | ------------ |
| Data Release 1      | 10,390 |
| Data Release 2      | 104,923 |
| Overlap between 1 & 2 | 10,385 |

## WGS Telseq Pilot

Almost 90% of the samples in the Telseq pilot are also included in Data Release 2. Lists of Telseq analyzed samples that overlap with Data Release 1 &2 are available in the telomere collaboration bucket.

| Description | Sample count |
| ----------- | ------------ |
| Telseq Pilot | 78,385 |
| Data Release 1 overlap | 6,043 |
| Data Release 2 overlap | 69,823 |
| Overlap with both releases | 6,042 |
| No overlap | 8,561 |

# Philosophy for releasing data

As we start to get a handle on the technical challenges of releasing large datasets, it's worth taking some time to also consider how we release data. Our primary objective is to deliver whole genome sequencing data to Million Veteran Program researchers. When considering how to deliver data, the context is just as important as the content. Questions for us to consider regarding delivery:

- Where should data be made available?
- In what format(s) should it be available?
- How should data be filtered for quality before delivery?
- Should data be filtered for quality?
- What kinds of questions are researchers trying to answer with this data?
- How often should we deliver data?
- What kinds of data should we deliver?
- Should different kinds of data be delivered separately or together?

This is also something the folks at All of Us are thinking about and you can find their Data Roadmap here: [https://www.researchallofus.org/data-tools/data-sources/](https://www.researchallofus.org/data-tools/data-sources/).

## Brief thought: naming conventions

Instead of naming things based on the number of elements, I recommend naming things based on iterations (e.g. "wgs-data-release-1", "telomere-pilot", "telomere-study-2"), starting from (1), where a "pilot" indicates that the purpose of the study is research and development rather than releasing data to MVP researchers.

A core tenet of "big data" is scalability and a resource that is named according to how many samples it is designed for is inherently not scalable. Every production resource you create should be designed to scale down to 1 and scale up to 1 million. If not, then you are just going to have to remake it every time we want to add more samples, which is always.

# Discuss
Join the discussion on our <ins>[GitHub](https://github.com/orgs/va-big-data-genomics/discussions/23)</ins>!
