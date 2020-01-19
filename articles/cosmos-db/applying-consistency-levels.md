---
title: Applying consistency levels in Azure Cosmos DB
description: Applying the right consistency levels is a matter of what business question you are trying to answer.
author: seanwhitesell
ms.author: 
ms.service: cosmos-db
ms.topic: conceptual
ms.date: 01/18/2020
---
# Applying consistency levels in Azure Cosmos DB

## Replication

Azure Cosmos DB uses snapshot isolation to allow for data to be updated with the least amount of impact. Other databases may use table or row locking mechanisms. By utilizing snapshot isolation the read is able to retrieve the latest version of that data. But, the read may pull from a replica that has to be updated. Write operations come to the Leader replicas of each replica set. As data is replicated to the other replicas in the set it may also be replicating to other replica sets in other regions if using multiple regions.

The replication occurs as fast as possible. Inside a single region replication is nearly zero latency. Of course there is some latency as data goes across the network to other servers housing the replicas. Replication latency to other regions does take a longer time. We are now at a time when the speed of light has an impact on our architecture. From one region to another there is approximately 10 ms of latency per 1,000 miles. This is due to not only the speed of light but also network hops and processing.

Leveraging [Consistency Levels](consistency-levels.md) starts with understanding replication and the needs of the business functionality applied to Cosmos DB. Replication and details of replica sets are described here: [Global data distribution with Azure Cosmos DB - under the hood](global-dist-under-the-hood.md). 

## Consistency Levels in Action

In our scenario, a client application is updating a document multiple times a second. The Read operations of that same document are also occurring multiple a second. Notice that the chosen consistency level affects the Read operations of that same document. Read operations are only affected when the document might be updated by a Write operation. Knowing when or how often a document might be updated is entirely dependent on the architecture of the application(s) using that instance of Cosmos DB.

- **Eventual**: As a document is updated and replicated to all replicas, a Read using Eventual consistency may return the updated version but more likely return a previous version. Imagine a Read occurs a week after a document as been updated. In this case the Read will return the most up-to-date version. Now, if that Read was only 2ms after the Write, then the returned version is most likely coming from an replica not yet updated with the new version. And, given our scenario of multiple versions of a document in such a small time frame, the returned versions may be out of order compared to the order of the updates. For time critical applications this not the best choice.

    Banking, for example, use Eventual consistency. Once a day all actions to an account (document) become consistent. That is, all actions to remove and add monies are applied and that information is recorded. Actions after that posting time are applied at the next posting operation.

- **Consistent Prefix**: This consistency level is much like Eventual except that the versions returned in the Read operations are in the same order as updated. The returned version may not be the most up-to-date version compared to the latest update but they are in order. And, when reading the document multiple times the version returned is never older than one previously returned.

- **Session**: The same details apply to Session as Consistent Prefix. The difference here is simply that the client connection that updated the document can subsequently perform a Read operation and the updated version is returned. Other client connections doing a Read may have an older version returned. Notice this is based on the client connection, not the application. This is done by using a Session Token. Once an update is done a Session token is returned and Read operations for that document must use that Session Token on the same connection.

- **Are we in sync yet?**: With these three consistency levels, Eventual, Consistent Prefix, and Session, there is no guarantee when the replicas have synchronized. The Azure infrastructure makes every attempt to replicate data as soon as possible. However, there is no implied guarantee replicas are synchronized in 10 ms or 10 minutes. When working with globally replicated data this detail matters. If an application cannot tolerate out of sync data in one region compared to another in a reasonable time frame then the next two consistency levels are better options. 

- **Bounded Staleness**: 


- **Strong**: 


## Key Take-Aways

With Azure Cosmos DB you must apply a *default consistency level* on the account. This means that no Read operation can use a consistency level that is stricter than that default. The key here is that each Read connection in code can use a different consistency level than the one used to Write the data. If, for example, the *default consistency level* is set to Bounded Staleness and data is being retrieved for Data Analytics, then a consistency level of Eventual can be used since there is no need for the most up-to-date information. Data Analytics is usually working with data at least hours old. Most of the time it is with Year-to-Date or Month-to-Date type of reporting.

And just as Data Analytics can use one consistency level for their needs, another can be used for a different need against the same data. It's about the importance of the Read compared to when the data has/is updated.


