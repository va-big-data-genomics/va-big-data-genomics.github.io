---
layout: post
title:  "Sustaining bioinformatics infrastructure: lessons from PBOT"
date:   2023-01-12 15:10:02 -0800
author: Paul Billing-Ross
categories: jekyll update
---
![Hawthorne Bridge in Portland](https://images.unsplash.com/photo-1590866249433-0310439fdab9?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=1170&q=80)
<p align = "center">
<font size = "3">
Photo of the Hawthorne Bridge in Portland by Ryan Manwiller on <a href="https://unsplash.com/photos/he5iQ3hgbv8">Unsplash</a>.
</font>
</p>

## Introduction
Our group is responsible for building and maintaining robust cloud based infrastructure to manage, process, and analyze large scale genomic datasets. As a small group in a rapidly evolving field, we are under constant pressure to expand our offerings and generate new datasets. Our challenge is to establish a framework for adding new features while also sustaining existing ones, all while the size of our datasets gets steadily larger. By taking lessons from physical systems, we can learn to better manage our digital ones.

## Focus on The Process
A lot of the internal and external pressure on our group is in regards to generating data, because our stakeholders need data to generate results, and all of the incentives in academia are structured around results. Nobody awards grants for maintenance. It's tempting to conform to these pressures and choose the shortest path towards generating data and getting results. However, the shortest path to results is to forego establishing robust, well documented, reproducible methods; which is why so much scientific research is not reproducible.

And while the "[crisis](https://www.nature.com/articles/533452a)" of reproducibility is an abstract one for most researchers, it is of immediate concern to us because our job is to repeat these methods to generate larger and larger datasets ad infinitum. And the bigger the data, the greater the strain on the scalability of our methods.

### A $4 billion backlog
We don't build roads but we do build infrastructure, and when I think of the importance of building and maintaining robust infrastructure, I think of my local Portland Bureau of Transportation (PBOT). PBOT currently has a [$4.4 billion maintenance backlog](https://bikeportland.org/2022/11/02/whats-behind-pbots-4-4-billion-street-maintenance-backlog-excuse-366371). This has all kinds of negative consequences:

* Rejection of new projects
* Frustration of PBOT workers who can't address problems
* Frustration of residents who want good infrastructure

And every year the maintenance issues go unresolved, the burden of using their infrastructure increases, as does the cost of fixing it, creating a negative feedback loop that will take massive investment to resolve.

### How did they get there?
Well, it didn't take long. In 2005, the maintenance backlog only amounted to $28 million. Official site multiple causes:

* Inadequate maintenance funds for expensive infrastructure
* Lack of political will: maintenance isn't sexy
* Bigger and heavier vehicles
* Federal grant programs don't pay for maintenance (Hello, NIH!)

How are they going to fix it? Not easily and not soon. So, what lessons can we translate to bioinformatics infrastructure?

# Lessons for bioinformatics

* Focus on maintaining existing systems from the outset. Once you get into a hole, it's difficult to get out.
* Regulate usage of your infrastructure to sustainable methods. Less repeatable means more expensive to maintain.
* Budget maintenance into your capacity for projects and assume that every resource requires maintenance to address things like bigger datasets, new types of data, software updates, and cloud platform updates.
* Measure the time you are spending on each project. How does actual time spent match your expectations?

## Establish a Strategy and Specifications
Expanding features and services with minimal labor cost is a significant challenge. Another, more positive example from PBOT demonstrates a strategy for doing so.

One of the projects that has been hamstrung by PBOT's budget woes is the expansion of bike lanes. But, in 2022, they got a new section of [bike lanes](https://bikeportland.org/2022/10/28/a-global-corporation-paid-for-and-built-protected-bike-lanes-on-nw-front-ave-366380) built at no cost to the city. How did they do it? They didn't. They provided specifications for the bike lanes and a local corporation hired contracts to install them near their headquarters. Why they did it somewhat questionable (people had been parking RVs along that strip of road), but ethics aside, the city got new infrastructure at no cost. Again, how does this apply to digital infrastructure?

### Publish specifications for extending infrastructure
If others want to extend our infrastructure with new algorithms or features, let them. Ideally, provide tutorials for development, testing, and validation. Just make sure their additions meet the specifications you have established. Introducing variability into the system will make it more complicated and harder to maintain. Can you write automated tests to validate that new features meet specifications? Teach people to fish, rather than giving them fish.

### Establish a strategy for expanding infrastructure
The new bike lanes weren't placed arbitrarily, they were added in accordance with PBOT's Bicycle Plan for 2030, an overarching plan that includes a map of bike lanes intended to be added over a 20 year period.

When PBOT, an institution tasked with advancing the public good, adds a bike lane they aren't just creating a path; they are indicating to the public that they should bike there. It signals to the public that "We experts have studied traffic patterns and added this path because it is optimal route for cycling to some place." If people use the path and find that it is not an ideal route, it erodes public confidence in the entire institution.

As a data generator, our goal is to provide high quality datasets to all researchers in the [Million Veteran Program](https://www.research.va.gov/mvp). By adding infrastructure to generate a new dataset for researchers, we are tacitly endorsing that dataset; that we have vetted the algorithm that generated it, validated the results, and are confident in the quality of the data. By performing this due diligence we allow researchers to focus on analyzing the data rather than validating it.

By adhering to these principles, we can transition from research-grade to enterprise-grade infrastructure, and establish a foundation that will support new features and ever expanding data.
