---
layout: post
title:  "#24 (Re)thinking Trellis messaging"
date:   2023-06-016 10:00:00 -0800
author: Paul Billing-Ross
categories: jekyll update
tags: [Mermaid]
mermaid: true
---

## Background
Trellis is a distributed system of serverless functions, operating independently to automate variant calling and quality control procedures. Because each function is independent, a robust messaging system is essential to keep functions synchronized with each other and ensure the health of the system as is is developed.

<div class="mermaid">
    graph TD
        gcs[Cloud Storage] --> create{{create-blob-node}}
        create --> db-query{{db-query}}
        db-query --> check-triggers{{check-triggers}}
        check-triggers --> db-query
        db-query <--> neo4j[(Neo4j database)]
        db-query --> job-launcher{{job-launcher}}
        job-launcher --> db-query
        job-launcher --> api[Life Sciences API]
        api -- dataObject --> gcs
</div>
Fig 1. Architecture diagram describing the flow of information between core Trellis serverless functions and other essential systems. The hexagonal nodes are Trellis functions. The boxes represent cloud platform systems and the cylinder a database.

In the most recent Trellis release, each function implemented it's own methods for parsing and sending messages to other Trellis functions via Cloud Pub/Sub. With newer functions, I had created a a `TrellisMessage` class to parse incoming Pub/Sub messages in a uniform manner, but the code for that class was still written separately into every function. So even though I added the intention of handling messages consistently, the implementation allowed for different versions of the class to become out of sync with each other in different functions.

A major change in the Trellis v1.3 update is moving most of the Trellis logic into a Python package so that it can be imported and used by any Trellis function, in a uniform manner. The logic coded directly into functions should either unique to the operation of that function or boilerplate that is necessary for all serverless functions.

## Trellis Messaging
An essential module of the Trellis package is messaging. A major challenge in conveying information between functions is that it has to all be by plain text. In a monolithic application architecture you can use a programming language to create instances of complex class objects and then pass them between different parts of the system, because the entire system is operating in the same environment. And becaue they all exist in the same environment, they can all respond to changes in the environment. In our distributed system, every function exists in its own environment, so every piece of relevant information must be specifically delivered to the function that needs it.

The upside of this modularity is scalability and robustness. Because monolithic architecture exist in the same environment, all of the computers they run on must be networked together. Much research computing happens in these kinds of high performance computing clusters that require large, complex, and expensive infrastructure located in a dedicated data center. And thus the scalability of the system is limited to the computer resources at that site and can lead to different elements of the system competing for resources and restricting each other. It also means that failures in the environment can impact the whole system. In a distributed system, every element can run at a different site and in a different environment, scaling upwards without competing for resources. Distributed systems can also be more resilient because failures in one environment will not affect environments of other elements.

Some of these theoretical differences do not necessarily bear out in the real world. For instance, all Trellis functions run in the `us-west` region of Google Cloud Platform which corresponds to a single data center located along the Columbia River in Dalles, Oregon. So there is a technical limit to how big our functions can scale there, but it's so big as to to essentially be limitless. And if the data center fell into the river all Trellis functions would stop working. But, I could also set up Trellis functions to work in multiple regions, across multiple data centers. I could also set them up to work on different cloud platforms, because the only connection they need is the ability to communicate with other functions via text.

![Dalles data center](/assets/2023-06-16/the-dalles-cooling-towers.jpg)
Fig 3. Image of The Dalles data center in Oregon, courtesy of [Google Photos](https://www.google.com/about/datacenters/gallery/#!#the-dalles-cooling-towers). 

So, then the problem became, how should these functions communicate. To enable scalability of the system, in terms of being able to add new functions, it was important that messaging be standardized. Standards provide developers and would-be-developers a better understanding of how to interact with the system and also decrease maintenance demand and potential for bugs. The entire system is built around a graph database and interacting with graph events. Specifically, task automation is achieved by database triggers that initiate events in response to new nodes or relationships being added to the database. And bioinformatics jobs are configured by mapping metadata from graph objects into a `dsub` arguments.

Because all Trellis operations are graph based, it seemed appropriate that a primary role of messaging would be to communicate graph data by text. Initially, I conceived of solving this problem by creating two separate classes, `QueryResponseWriter` and `QueryResponseReader` to write graphs to text, in [JSON](https://en.wikipedia.org/wiki/JSON) format, and then read them from JSON back into a graph. I chose to do this because I would be initializing these objects with different types of data. The `QueryResponseWriter` would be getting `neo4j.Graph` data from the database, while the `QueryResponseReader` would be getting JSON formatted data from a Pub/Sub message, so I would need different initialization functions. A couple problems with this model were that...

1. I kept on thinking of query responses as being the objects that functions would interact with, but in my model it was actually the actions of writing and reading that would be modelled as classes. So, there was just an awkward mismatch of data and concept.
2. The structure of the graph was still implemented in two separate classes that could become out of sync with each other.

<div class="mermaid">
flowchart TD
    neo4j(neo4j) --> graf[neo4j.Graph]
    neo4j --> summary[neo4j.ResultSummary]
    graf --> writer[QueryResponseWriter]
    summary --> writer
    writer --> json[JSON]
    json --> reader[QueryResponseReader]
    reader --> graf2[neo4j.Graph]
    reader --> summary2[neo4j.ResultSummary]
</div>
Fig 3. Diagram describing initial model of messaging that used separate classes to write and read graph data.

To address these issues, I developed a new model where I would use a single `QueryResponse` class to represent graph data generated from database queries and use separate helper functions to to convert from `neo4j.Graph` to JSON and then back to a `neo4j.Graph` object again. This way, the query response object would always be initialized using JSON data and instances of the query response would be identical on the side of the function that was sending the message as well as the side that was receiving. Additionally by moving the graph-to-JSON logic out of the message class, I left open the possibility of integrating Trellis with other graph/databases by implementing different helper classes while still using the same message class.

<div class="mermaid">
flowchart TD
    neo4j(neo4j) --> graf[neo4j.Graph]
    neo4j --> summary[neo4j.ResultSummary]
    graf -- trellis.translate_graph_to_json --> jsonGraph[json_graph]
    summary -- trellis.translate_summary_to_json --> jsonSummary[json_summary]
    jsonGraph --> qr1[trellis.QueryResponse]
    jsonSummary --> qr1[trellis.QueryResponse]
    qr1 -- Pub/Sub --> qr2[trellis.QueryResponse]
    qr2 -- json.loads --> Dict
    qr2 -- trellis.translate_json_to_graph --> neo4j.Graph
</div>

Fig 4. Diagram of Trellis messaging workflow using a single message class with helper functions.

I also considered whether it was worthwhile for downstream functions to convert messages back into the `neo4j.Graph` format that the Neo4j driver returns database results in. On one hand, all of the graph data is available as a dictionary attribute of the `QueryResponse` class. On the other hand, a dictionary is a generic data structure for holding all kinds of data, while the `neo4j.Graph` class is designed specifically for holding graphs, and reconstituting the data as a graph validates it's graph structure. I use the Neo4j class because we are using a Neo4j database, but we could also use any other library that was designed to handle graph data. In fact, there are probably other ones that are better suited for programmatically interactin with graphs in Python. For now though, I'm just going to continue using the Neo4j one.

## Grouping results
Another issue I encountered was that of grouping results. In some cases, I will want to delivery all the result of a query to a function, together. This is the case for launching the GATK $5 pipeline, where I use a query to find all the Ubams for particular sample and pass them as input to the job. In other cases, I may want to split each query result to send to a different function. For instance, finding every GVCF that does not have Vcfstats data and requesting a job each of them. The Neo4j driver has a built in method for splitting and returning results, but it is as instances of `neo4j.Record` rather than `neo4j.Graph`. A question I had was whether I could reconstitute a `neo4j.Graph` object from record data or whether I would implement custom splitting logic. After some testing I found that I was and that by adding another helper class I found also be able to integrate `neo4j.Record` data into the new model.

<div class="mermaid">
graph TD
    neo4j[(Neo4j database)] -- aggregate --> grf(neo4j.Graph)
    neo4j -- split --> rec1(neo4j.Record)
    grf -- trellis.graphToJson --> grfjson[Graph JSON]
    rec1 -- trellis.recordToJson --> grfjson
    grfjson --> response(trellis.QueryResponse)
    response -- trellis.QueryResponse.convert_to_json --> respjson[Query response JSON]
    respjson --> pubsub{Cloud Pub/Sub} 
    pubsub --> respjson2[Query response JSON]
    respjson2 --> response2(trellis.QueryResponse)
    response2 -- jsonToGraph --> grf2(neo4j.Graph)
</div>
Fig 5. Diagram of grouping graph results in the new messaging model.


# Discuss
Join the discussion on our <ins>[GitHub](https://github.com/orgs/va-big-data-genomics/discussions/27)</ins>!