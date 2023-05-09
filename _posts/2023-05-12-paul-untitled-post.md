---
layout: post
title:  "# (n) pbilling untitled"
date:   2023-05-12 10:10:00 -0800
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

## New model architecture

A cloud function named "add-study-participants" will listen for new objects added to a "trellis-studies" bucket. When a new object named "*-trellis-study.yaml" is added, it will read the study configuration file in order to add the study information to the Trellis Neo4j metadata store. When users want to add a study to the database they will upload the YAML file as well as a CSV file with the relevant metadata for each study participant. This will include a "sample" column which will be used to map the (:Participant) nodes to their corresponding (:Sample) and (:Person) nodes.

Architecture diagram of relevant Trellis resources:

<div class="mermaid">
    graph TD
        gcs(Cloud Storage) -- CloudEvent --> add-study-participants
        add-study-participants -- QueryRequest --> db-query
        db-query -- QueryResponse --> db-triggers
        db-triggers -- QueryRequest --> db-query
        db-query -- Cypher --> neo4j((Neo4j DB))
        neo4j --> neo4j.Result --> db-query
</div>

Example study configuration object:

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

I want to be able to populate the (:Study) and (:Participant) nodes with metadata fields that are custom to each study, while still using (TODO: ADD LINK READTHEDOCS) parameterized queries, so instead of using a predefined query, the function will dynamically generate a parameterized query similar to as is done by the create-blob-node function (TODO: ADD LINK TO READTHEDOCS). All queries will be processed by the Trellis `db-query` function.

First, it will send a query to create study node that will look like the following.

```
MERGE (study:Study {name: $name})
SET study.notes = $notes, 
    study.type = $type, 
    study.lead = $lead 
RETURN study
```

Then, it will send a set of queries to create the participant nodes. If I am understanding the <ins>[Pub/Sub documentation](https://cloud.google.com/pubsub/quotas)</ins> correctly, the maximum size of a Pub/Sub message is 10MB. Because we want this function to be scalable, the function will split the data from the CSV into multiple query request messages if it exceeds a predefined threshold (e.g. 9MB).

Example query to add study participants to database:

```
UNWIND $data AS participant_entry
MERGE (participant:Participant {
        study: $studyName,
        sample: participant_entry.sample
})
SET participant.telomereLengthEstimate = toFloat(participant_entry.lengthEstimate
RETURN len(participant)
```

The result of the participants query will trigger a query request to connect the (:Participant) nodes to their corresponding (:Sample) and (:Person) nodes, using the "sample" field. 

<div class="mermaid">
    sequenceDiagram
    add-study-participants ->> db-query: QueryRequest: createStudyNode
    db-query ->> db-triggers: QueryResponse: createStudyNode
    add-study-participants ->> db-query: QueryRequest: createParticipantNodes
    db-query ->> db-triggers: QueryResponse: createParticipantNodes
    db-triggers ->> db-query: QueryRequest: relateParticipantsToSamples
    db-query ->> db-triggers: QueryResponse: relateParticipantsToSamples
    db-triggers ->> db-query: QueryRequest: addStudyParticipantCount
</div>

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
