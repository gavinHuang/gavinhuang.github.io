---
layout: post
title: "the tale of two pipelines"
date: 2021-11-05 19:11:26 +1100
comments: true
categories: Data
tags: Data
---
### Background
I have came across a few articles that clear some of misconceptions around advanced analytics. Two of them, [DataOps is NOT Just DevOps for Data](https://medium.com/data-ops/dataops-is-not-just-devops-for-data-6e03083157b7) and [The Next Big Challenge for Data is Organizational](https://locallyoptimistic.com/post/the-next-big-challenge-for-data-is-organizational) has been my favorite.

I cherry-picked some of ideas from them, try to apply them to my work context, turns out it helped a lot to set my expectation correctly and align my plan with the overall organizational vision and strategy.

Here are some punch lines:

1. The fundamentally different between data warehousing and the advanced analytics (or machine learning application) is that the first one is value pipeline and the later is the innovation pipeline. Some other finding is derived from that perspective.

2. It's the organizational factor that challenge data or advanced analytics projects. 

The first one set the tone for the second one, while the second one will make fundamental change to your organization's data approach ( sometimes can be right v.s wrong, rather than just different.).

<!--more-->

### Symptom of confusion

You can easily find people from variety of background in advanced analytics teams. For example, a data engineer could be a former DBA, or a application (mostly backend) developer, or a report developer (for a specific BI tool). When talking about building a modern data platform/application/pipelines, people tend to picture it based on their experiences in previous domain. Therefore, to some extend, to build the so called modern data platform is just using modern tools to build a data warehousing platform (for a former data modeler), or portfolio of applications with some common data set (for a former application developer). 

The discrepancies derived from the different understanding not only lead to complete different system design and process, but also the design of the team structure. A data warehousing-centric approach might force developers to develop data application in data warehousing system and vice versa.

Both opinions got their supports, especially from data solution vendors. They tend to claim their product can solve most (if not all) data problem, so they tend to push for reference architecture that centered with their solution. For example, Snowflake, in my opinions is a cloud-based data warehousing platform, propagating data platform that are centered around snowflake; Databricks, the famous spark company, push for the concept of "lakehouse".

The discrepancies not only lead to different system architecture, but also determines the different team structure.  The "data warehousing" supporters tend to build a big team who not only control the production-harden warehousing system, but also provide essential support for other critical teams like data governance, data literacy, etc.  The "Data application" supporters tend to play down the important of data warehousing, deem the warehousing team as the data source for reporting. the words often heard in debating are: real-time, patterns, MLOps, operationalization, etc.

### The Right Way?

So what is the right approach for a modern data platform and team? As always, there is no correct answer to the question. My current employer is a brick-mortar retail company, just a few years on its advanced analytic journey.  For company like this, any technical solution that claim to be the "industry standard" which covers the most trendy tech-stack that you could possibly think of, is suspicious. According to Conway's law, the team structure actually is a reflection of the architecture (or the opposite for an established team), Therefore, a simple but robust system requires less complicated team structure to carry out and quick to refactor to the change.

#### 1. Architecture

According to [DataOps is NOT Just DevOps for Data](https://medium.com/data-ops/dataops-is-not-just-devops-for-data-6e03083157b7) , there are two different kind of pipelines: the value pipeline and the innovation pipeline. Among many differences between the two, the most significant one in my opinion is the different variables in the system: the code and the data. Value pipelines deal with data change while the code mostly stable. Without further explanation, it immediately appear to me the value pipeline means data warehousing. Based my experience of data warehousing project, the every day problem is caused by the data change, while the code relatively stable.

On the other hand, the innovation pipelines are more likely to be an application whose daily job is to deal with business logic change with code change (database is a tool to help facilitate the change). For innovation pipeline, the paramount principle is the quickly react to external change and apply the change as quick as possible.

#### 2. Team Structure

[The Next Big Challenge for Data is Organizational](https://locallyoptimistic.com/post/the-next-big-challenge-for-data-is-organizational/)  believe the key to design a team structure is the contracts between teams. A good example is the front-end team and back-end team, while their skill set are different, the contracts between them to make system works are clean: the API (mostly restful API). To apply the same rule to modern platform, what is the contract a data warehousing team should adopt? Same question apply to data applications.  

Assume there is a protocol or contract can be drawn somewhere within the bigger group, then a team structure can be designed around that. 

### The Blurring Boundary

Again, there is no silver bullet to every organization, especially when data domain is still evolving dramatically. What I can see is new technologies such as dbt, snowflake and other workflow (orchestration) tools are redefine the boundary between data warehouse and data application. People from different background is quickly adopting to the new industry pattern. What a modern data team and platform should looks like is still to be answered.