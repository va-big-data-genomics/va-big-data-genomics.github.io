---
layout: post
title:  "#39 Seven years of genome quality control"
date:   2023-12-15 10:05:00 -0800
author: Paul Billing-Ross
categories: jekyll update
faq:
- question: "What was the first quality control pipeline you inherited?"
  answer: "The [Bina processing pipeline.](https://github.com/StanfordBioinformatics/qualia/blob/master/bina-processing.md)"
- question: "Where can I find Greg's BigQuery quality control paper?"
  answer: "[Cloud-based interactive analytics for terabytes of genomic variants data.](https://academic.oup.com/bioinformatics/article/33/23/3709/4036385?login=false)"
- question: "How did you track the status of genomes before Trellis?"
  answer: "A [spreadsheet](https://docs.google.com/spreadsheets/d/1-j2q816XnF7D7gktFRsxyA9LMIE8_c7beHh6xVTlxMs/edit#gid=0)."
- question: "Where is the 2018 presentation to Mike Snyder?"
  answer: "In the GBSC presentations [folder](https://docs.google.com/presentation/d/1e37lkla4E8ZVmLQz9aq--OEEiFt1XYeUaNwpbANuZRU/edit#slide=id.g465b0ec56a_4_0)."
---

When I think about modelling data I think about two things: where did you come from and what can you do? This is my last post as a member of this team. Next year I'll be setting out on my own professional path. You can read my previous posts to see what I can do. This post is about how I got here. And in understanding my own provenance, hopefully you can better understand the decisions I've made.

I joined the Million Veteran Program team 7 years ago, in the fall of 2016, when it was Greg and Cuiping doing the work and Phil and Somalee managing. I had joined Stanford a little over a year previous and had transitioned the sequencing center pipeline from on-premise to DNAnexus. It had gone pretty well and I had time to spare.

Greg and I had became friends during our undergraduate time together working in Dr. Sudhir Kumar's lab at ASU, and he was the reason I had found the Stanford job listing. When I interviewed at Stanford, he was working on converting genetic algorithms to SQL so that they could be run on large datasets living on BigQuery. By the time I joined the project, they were writing [the paper](https://pubmed.ncbi.nlm.nih.gov/28961771/), using a set of 475 whole genomes. 

Greg was heading off to do a PhD, so Cuiping tasked me with running his quality control pipeline on (500) more genomes that were incoming from Personalis. This might be the first [document](https://docs.google.com/document/d/1kzvcDBoMJ_CHVwfxXQlCrFxkcfWJwWDqQkFqlDCd31Q/edit) I created as part of the MVP program. I did not have particularly nuanced ideas about documentation back then, so I just created a Google Doc and dumped all my meeting notes into it. I used that document for (2) years.

To call it a pipeline is maybe a stretch. There were a lot of manual commands and not a lot of pipe. In order to initiate each quality control task, I would create a text file with a list of the paths to each input (Bam, Fastq, etc.) and pass that to a script running in a Docker container. It was not an incredibly smooth process.

Following the batch of (500) genomes, we continued receiving genomes from Personalis until we had a set of (1902) genomes. My initial innovation was to create a spreadsheet to track the quality control status of genomes as we received them in different batches. This innovation did not result in the drastic quality of life improvement I was hoping for. One issue was that I was only tracking the number of genomes that had passed each quality control step. So if, for any reason, any sample had failed any of the 17 steps involved in running quality control tasks and then importing them into BigQuery, I had to go back and run manually `ls` commands to try to figure out which samples had failed which steps and *then* perform the troubleshooting to figure out why they had failed.

I think it was during an initial interview with a tech lead at Phylagen that I was describing this whole painful process when he amiably asked "so, how would you do this differently now?" And that is when it occurred to me that these are the kinds of problems people use databases for. I was importing the data into a database but it had never occurred to me that the process for importing data into a database could *also* be tracked in a database. A meta-database, if you will.

So, from there I set out to figure out how to track bioinformatics workflows in a database to better keep track of which samples had undergone which quality control steps. When I look at a bioinformatics workflow I see a graph, so my first attempt was to model workflows, as graphs, in a database. Initially, I tried building my own graph database using references in a document store and a home brewed graph browser I prototyped with the [d3 library](https://d3js.org/), before realizing that “graph database” was already a thing (though just barely), and switching to Neo4j.

In 2018, Personalis began delivering a new batch of (9000) whole genomes. This would be the first dataset managed by Trellis. Even though these are MVP genomes they are partitioned from the rest of the MVP genomes by virtue of 1) they were processed slightly different by Personalis and 2) they were modeled using an older Trellis database schema that is not in use. The database still exists though; you can find the virtual machine named "neo4j-trellis-wgs9000" in our production project. 

By October 2018, Jina Song had joined the team and Trellis was up and running. According to our presentation to Mike Snyder, Trellis had performed about 200,000 QC tasks for 3,740 samples. We had a lot more tasks to perform back then because Personalis delivered one bam per sample per chromosome. As a sidenote, we started receiving genomes before Trellis was operational, so Jina's first project on the team was learning how to run my hand-crafted pipeline by hand. It is a credit to her strength of character that she did not immediately quit.

The second phase of Personalis genomes began arriving in early 2019. This is the much larger tranche of 100,000+ genomes that are tracked in the production Trellis Neo4j metadata store and which were shared with researchers in data releases 1 and 2. With the lessons I learned from graph modeling of phase 1 data, I overhauled the Neo4j schema for phase 2. Because the schemas were so different, I used an entirely different database instance for phase 2 with the intention of reimporting all the metadata from the phase 1 genomes into the new database (Note: this has still not occurred).

So now I had over the keys to a new group that is already making impressive progress. Particular congratulations to Prathima who presented her work analyzing telomere length and Joe who has orchestrated our second and largest release of Million Veteran Program genomes. And if you feel like progress is slow, consider that it took us (5) years to release the first batch of genomes and only (2) to release the second batch with 10 times as many. That's a pretty good trend. 

Finally, in the past couple years I have prioritized documentation. There are lots of reasons I think you should document your process but because this is a somewhat sentimental post I'll emphasize one: this is your story and I would certainly like to read it.
