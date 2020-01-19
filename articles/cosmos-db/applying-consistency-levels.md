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

Azure Cosmos DB uses snapshot isolation to allow data to be updated with the least amount of impact. By utilizing snapshot isolation the read is able to retrieve the latest version of that data. But, the read may pull from a replica that has to be updated. Write operations come to the Leader replica of each replica set. As data is replicated to the other replicas in the set it may also be replicating to other replica sets in other regions if using multiple regions.

The replication occurs as fast as possible. Inside a single region replication is nearly zero latency. Of course there is some latency as data goes across the network to other servers housing the replicas. Replication latency to other regions does take a longer time. We are now at a time when the speed of light has an impact on our architecture. From one region to another there is approximately 10 ms of latency per 1,000 miles. This is due to not only the speed of light but also network hops and processing. Applications may see higher latency due to the extra processing to send/received that data. 

Leveraging [Consistency Levels](consistency-levels.md) starts with understanding replication and the needs of the business functionality applied to Cosmos DB. Replication and details of replica sets are described here: [Global data distribution with Azure Cosmos DB - under the hood](global-dist-under-the-hood.md). 

## Consistency Levels in Action

In our scenario, a client application is updating a document multiple times a second. The Read operations of that same document are also occurring multiple a second. Notice that the chosen consistency level on the connection affects the Read operations of that same document. Read operations are only affected when the document might be updated by a Write operation. Knowing when or how often a document might be updated is entirely dependent on the architecture of the application(s) using that instance of Cosmos DB.

Our scenario is of a manufacturing company. They have assembly lines where their special products are produced. As they are produced and move along on the assembly line they pass sensors. The sensors update information in the Cosmos DB. In the office, different departments use an application that relies on information gathered in the shop. For our examples, the Cosmos DB account default consistency level is set to Strong. Later in this document we will go over why you cannot use Strong in a certain configuration with globally replicated data.

- **Eventual**: As a document is updated and replicated to all replicas, a Read using Eventual consistency may return the updated version but more likely return a previous version. Imagine a Read occurs a week after a document as been updated. In this case the Read will return the most up-to-date version. Now, if that Read was only 2ms after the Write, then the returned version is most likely coming from an replica not yet updated with the new version. And, given our scenario of multiple versions of a document in such a small time frame, the returned versions may be out of order compared to the order of the updates. For time critical applications this not the best choice.

    Banking, for example, use Eventual consistency. Once a day all actions to an account (document) become consistent. That is, all actions to remove and add monies are applied and that information is recorded. Actions after that posting time are applied at the next posting operation.

    Eventual consistency works well with Data Analytics and Reporting. The information needed does not have to be up to the split second accurate. Reports like Year-to-Date and Month-to-Date usually use information that is accumulated from other data and not required to be up to that very second.

- **Consistent Prefix**: This consistency level is much like Eventual except that the versions returned in the Read operations are in the same order as updated. The returned version may not be the most up-to-date version compared to the latest update but they are in order. And, when reading the document multiple times the version returned is never older than one previously returned.

    Imagine our pretend manufacturing company has a manager that asks the question "How many products have been made so far per line in this hour?" The sensors are updating information in the document. This is where the application can use the Consistent Prefix consistency level. If the application was to Read multiple times inside the hour the expectation is the answer will never be lower than a previous Read. Using Eventual would allow for a accurate answers that are out of order. The application cannot use Session since it did not update the information.

- **Session**: The same details apply to Session as Consistent Prefix. The difference here is simply that the client connection that updated the document can subsequently perform a Read operation and the updated version is returned. Other client connections doing a Read may have an older version returned. Notice this is based on the client connection, not the application. This is done by using a Session Token. Once an update is done a Session token is returned and Read operations for that document can use that Session Token even with a different connection.

    When an application's business logic is dependent on the absolute assurance data has been written to the database Session is good choice. With local replicas and/or replicas other regions you may need to verify the replication has completed before the business logic can continue. The application can leverage [Custom Synchronization](how-to-multi-master.md). The premise is that one connection Writes data to a region. Another connection, using the Session token from the Write operation, performs a Read for that data. This Read can be from another region if using [Global Replication](distribute-data-globally.md).

- **Are we in sync yet?**: With these three consistency levels, Eventual, Consistent Prefix, and Session, there is no guarantee when the replicas have synchronized. The Azure infrastructure makes every attempt to replicate data as soon as possible. However, there is no implied guarantee replicas are synchronized in 10 ms or 10 minutes. When working with globally replicated data this detail matters. If an application cannot tolerate out of sync data in one region compared to another in a reasonable time frame then the next two consistency levels are better options. 

- **Bounded Staleness**: Now we are getting to a control of when data is guaranteed to be replicated. Measured in *K* and *T* where *K* is the number of operations and *T* is the amount of time in seconds. These values differ based on if only a single region is used or with multiple regions. With a single region and a Bounded Staleness setting of 5 seconds and 10 operations, then Reads that occur after 5 seconds, even if only 2 operations have occurred, will read the latest data at that point. Between the *K* and *T* it's whichever comes first.

    When multiple Reads occur, for the same document inside the window from the Write to, in our example 5 seconds or 10 operations, it is possible to receive stale data. The Reads will be in order just like Consistent Prefix and a Read can see its own Writes when using the Session Token. The table below shows the range for *K* and *T* for Single and Multiple Regions.

    |**Single Region**| | **Multiple Regions**|
    |-----------------|--|----------------------|
    |10 - 1,000,000|**K**|100,00 - 1,000,000|
    |5 seconds - 1 day|**T**|5 minutes - 1 day|

    Back to our scenario, a manager now asks another question, "How many products have we produced so far today?" This seems easy enough but this is when a developer asks a question to better understand the use case. "Do you need the most correct answer, or, just need an answer now?" The business logic in the application may not change but the Consistency Level chosen for the connection does change. When the answer is "just needed now" then Bounded Staleness is applicable. This implies that a Read that returns slightly stale data is allowed even if a Write has just occurred. This allows for an answer to obtained with minimal impact to the database. This is similar to someone starting with a count of products in one section but not the ones near the assembly lines. To include those would cause a disruption in the flow of the lines. When it comes to answering for the "most correct answer", Strong is the selected Consistency Level.


 Looking at the [Consistency Levels and Data Durability](consistency-levels-tradeoffs)  


- **Strong**: 



## How Replication is a Factor


## Key Take-Aways

With Azure Cosmos DB you must apply a [Default Consistency Level](how-to-manage-consistency.md) on the account. This means that no Read operation can use a consistency level that is stricter than that default. The key here is that each Read connection in code can use a different consistency level than the one used to Write the data. If, for example, the *default consistency level* is set to Bounded Staleness and data is being retrieved for Data Analytics, then a consistency level of Eventual can be used since there is no need for the most up-to-date information. Data Analytics is usually working with data at least hours old. Most of the time it is with Year-to-Date or Month-to-Date type of reporting.

And just as Data Analytics can use one consistency level for their needs, another can be used for a different need against the same data. It's about the importance of the Read compared to when the data has/is updated.


