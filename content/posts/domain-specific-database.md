---
title: Kubernetes is a domain specific database
date: 2019-11-23
tags:
  - kubernetes
---

We've all seen domain specific languages - regular expressions, CSS, LaTex. Every once of these are created by folks who need specific functionality on a regular basis. Some DSLs end up being hyper specific and only used by a single person or organization to improve productivity. Other DSLs, such as CSS, end up defining entire industries as they're a problem that everyone has. What does this have to do with Kubernetes?

It would be possible to think of Kubernetes as its own DSL, the YAML used has its own grammar and syntax. These get applied to the api-server and then domain specific things happen. This some of the real power behind Kubernetes' API. By thinking about what's happening behind the scenes a little more, it is possible to fundamentally understand what creating or modifying something will actually do to the system. More importantly, much of the magic goes away and composition of the pieces themselves becomes easier.

A regular three tier app normally consists of a frontend that users interact with, an API that machines interact with and a database to store state. Kubernetes flips this around - you're interacting directly with the database every time `kubectl apply` is run. Take a Pod as an example. Running `kubectl create` will add a record to the database that contains all the details required. Components observe this change, start to do work and then update the Pod's record based off what's happening in the system.

By flipping the story around, there are two major implications - end users are not necessarily meant to interact with API directly and the Kubernetes community doesn't need to build a solution for everyone.

<!--

# YAML is for machines, not users


# Everything is an implementation detail

Kubernetes does not need to own the implementation of the API and users are freed to 

- Anyone can implement a controller that modifies the behavior of a resource.
- The developer is required to manage a fundamentally asynchronous process instead of worrying about tradeoffs and assumptions being incorrect.
- Extending the database itself for hyper specific needs becomes trivial - just create another table.
- Components themselves can either be replaced one by one or thrown away entirely.
- As higher level tooling interacts with the resources themselves, users don't need to care about implementation and might not even know something else is happening under the covers.

Think of a table in a SQL database. These have a schema, can be queried and represent the state of the system. Sounds a lot like resources in Kubernetes, right? Each resource, such as a Pod, has a specific schema, can be queried or watched and ...

Distributed systems, container orchestration systems in particular, are asynchronous by nature. Once there are multiple hosts and processes interacting with the system it becomes distributed. 

Distributed systems, container orchestration systems in particular, are asynchronous. As asynchronous APIs are difficult for humans to work with, there is all kinds of research around how to make an asynchronous API - synchronous. Unfortunately, there are always tradeoffs. Sometimes the tradeoffs are around performance or latency, other times it ends up being what can be queried. In all these situations, it is only possible to have a strong abstraction layer when the problem being solved is exactly what the original designers imagined.

-->
