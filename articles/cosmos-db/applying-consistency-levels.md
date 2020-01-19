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

Azure Cosmos DB uses snapshot isolation to allow data to be updated with the least amount of impact. By utilizing snapshot isolation the Read is able to retrieve the latest version of that data. But, the Read may pull from a replica that has to be updated. Write operations come to the Leader replica of each replica set. As data is replicated to the other replicas in the set it may also be replicating to other replica sets in other regions if using multiple regions.

The replication occurs as fast as possible. Inside a single region replication is nearly zero latency. Of course there is some latency as data goes across the network to other servers housing the replicas. Replication latency to other regions does take a longer time. We are now at a time when the speed of light has an impact on our architecture. From one region to another there is approximately 10 ms of latency per 1,000 miles. This is due to not only the speed of light but also network hops and processing. Applications may see higher latency due to the extra processing to send/received that data. 

Leveraging [Consistency Levels](consistency-levels.md) starts with understanding replication and the needs of the business functionality applied to Cosmos DB. Replication and details of replica sets are described here: [Global data distribution with Azure Cosmos DB - under the hood](global-dist-under-the-hood.md). 

## Consistency Levels in Action

In our scenario, a client application is updating a document multiple times a second. The Read operations of that same document are also occurring multiple a second. Notice that the chosen consistency level on the connection affects the Read operations of that same document. Read operations are only affected when the document might be updated by a Write operation. Knowing when or how often a document might be updated is entirely dependent on the architecture of the application(s) using that instance of Cosmos DB.

Our scenario is of a manufacturing company. They have assembly lines where their special products are produced. As they are produced and move along on the assembly line they pass sensors. The sensors update information in the Cosmos DB. In the office, different departments use an application that relies on information gathered in the shop. For our examples, the Cosmos DB account has a single region with a default consistency level set to Strong. Later in this document we will go over why you cannot use Strong in a certain configuration with globally replicated data.

- **Eventual**: As a document is updated and replicated to all replicas, a Read using Eventual consistency may return the updated version but more likely return a previous version. Imagine a Read occurs a week after a document as been updated. In this case the Read will return the most up-to-date version. Now, if that Read was only 2ms after the Write, then the returned version is most likely coming from an replica not yet updated with the new version. And, given our scenario of multiple versions of a document in such a small time frame, the returned versions may be out of order compared to the order of the updates. For time critical applications this not the best choice.

    The banking industry, for example, uses Eventual consistency. Once a day all actions to an account (document) become consistent. That is, all actions to remove and add monies are applied and that information is recorded. Actions after that posting time are applied at the next posting operation.

    Eventual consistency works well with Data Analytics and Reporting. The information needed does not have to be up to the split second accurate. Reports like Year-to-Date and Month-to-Date usually use information that is accumulated from other data and not required to be up to that very second.

- **Consistent Prefix**: This consistency level is much like Eventual except that the versions returned in the Read operations are in the same order as updated. The returned version may not be the most up-to-date version compared to the latest update but they are in order. And, when reading the document multiple times the version returned is never older than one previously returned.

    Imagine our pretend manufacturing company has a manager that asks the question "How many products have been made so far per line in this hour?" This is where the application can use the Consistent Prefix consistency level. If the application was to Read multiple times inside the hour the expectation is the answer will never be lower than a previous Read. Using Eventual would allow for a accurate answers that are out of order. Using the Session consistency level does not help since it was not the application that performed the Write operation.

- **Session**: The same details apply to Session as Consistent Prefix. The difference here is simply that the client connection that updated the document can subsequently perform a Read operation and the updated version is returned. Other client connections doing a Read may have an older version returned. Notice this is based on the client connection, not the application. This is done by using a Session Token. Once an update is done, a Session Token is returned. And, Read operations for that document can use that Session Token even with a different connection.

    When an application's business logic is dependent on the absolute assurance data has been written to the database Session is good choice. For our scenario, the Human Resources department is adding a new employee to the system. The operations are done in steps. First a profile is made and saved to the database. Next they add insurance information. The person applying the information cannot proceed through the steps without the prior step being committed to the database. Using Session the application can Read and receive the latest version of the document just written.

- **Bounded Staleness**: Now we are getting to a control of when data is guaranteed to be replicated. Measured in *K* and *T* where *K* is the number of versions and *T* is the amount of time in seconds. These values differ based on if a single region is used or with multiple regions. With a single region and a Bounded Staleness setting of 5 seconds and 10 operations, then Reads that occur after 5 seconds, even if only 2 operations have occurred, will read the latest data at that point. Between the *K* and *T* it's whichever comes first.

    When multiple Reads occur, for the same document inside the window from the Write, in our example 5 seconds or 10 operations, it is possible to receive stale data. The Reads will be in order just like Consistent Prefix and a Read can see its own Writes when using the Session Token. The table below shows the range for *K* and *T* for Single and Multiple Regions.

    |**Single Region**| | **Multiple Regions**|
    |-----------------|--|----------------------|
    |10 - 1,000,000|**K**|100,00 - 1,000,000|
    |5 seconds - 1 day|**T**|5 minutes - 1 day|

    Back to our scenario. A manager now asks another question, "How many products have we produced so far today?" This seems easy enough but this is when a developer asks a question to better understand the use case. "Do you need the most correct answer, or, just need an answer now?" The business logic in the application may not change but the Consistency Level chosen for the connection does change. When the answer is "just need an answer now" then Bounded Staleness is applicable. This implies that a Read that returns slightly stale data is allowed even if a Write has just occurred. This is assuming multiple regions with a single master using a default consistency level of Strong. 

- **Strong**: This is the strictest of the Consistency Levels and has the highest latency compared to the others. As a Write operation occurs, it is not considered complete until it has been replicated across a Global Majority of the replicas. For a single region this means for 3 out of the 4 replicas must be updated. For multiple regions, for example 2 regions, then 6 out of 8 replicas must complete before the operation is considered complete. This increases the latency of the Write operation but provides the highest confidence of consistency across replicas across regions. To better understand latency with Consistency Levels, see [Consistency Levels and Data Durability](consistency-levels-tradeoffs).

    When the manager, in our scenario, asks for data that is the "most correct answer", then the consistency level for the connection is Strong. This is not allowing for any stale data as the business question requires the latest information. 
    
    If using a multi-master or *multi-write enabled* instance then Strong is not an allowed consistency level. Bounded Staleness is the strictest option available. If Strong was possible then the entire system would be considered a single point of failure. The possibility of stale data inside the *K & T* window allows for a more highly-available system.

## How Replication is a Factor

The table below shows the number of replicas required for Write and Read operations. It also shows an RU cost. Notice that with Eventual, Consistent Prefix, and Session, a Read only uses a single replica. That replica is not guaranteed to be the replica with the most up-to-date version. This assumes a Read within a relative time frame of a Write. Given normal operating conditions of Azure, replication is extremely fast. If using Session, the Read operation using a Session Token of a Write is guaranteed to read from the same replica.


|**Consistency Level**|**Quorum Reads**|**Quorum Writes**|
|--------------------|--------------|---------------|
|Strong|Local Minority (2 RU) |Global Majority (8 RU)|
|Bounded Staleness|Local Minority (2 RU)| Local Majority (8 RU)|
|Session|Single Replica using Session Token (1 RU)|Local Majority (8 RU)|
|Consistent Prefix|Single Replica (1 RU)|Local Majority (8 RU)|
|Eventual|Single Replica (1 RU)|Local Majority (8 RU)|

*Majority = 3 and Minority = 2

With the Write operations they are not considered completed until a majority of the replicas are done. With a single region this means 3 out of the 4 replicas must have the updated version. When using multiple regions, Strong requires replication to a global majority. With 2 regions, 6 out 8 replicas must be updated before the operation is complete.

Remember the setup of our scenario, at the top of the page, is setup to use a single region with a default consistency level of Strong. Now the company has decided to use global replication across three regions. Does this mean the latency is three times longer? Not necessarily. Replication to other regions are concurrent and not sequential. That is, they are replicated in parallel. The latency the application will see is that of the distance from the master to the region furthest away. For example, if the master region is in South Central US and is replicated to East US and UK South, the noticed latency is only that between South Central US and UK South.

## Handling Possible Data Loss

How are we to handle an outage? Azure along with all public and private cloud providers are not immune to forces that may impact the operations and availability of a data center. Network communication itself is error-prone no matter the distance or medium. Let's start by asking two questions. When and how can issues occur? And, how can we mitigate risks?

On the page [Consistency Levels and Data Durability](consistency-levels-tradeoffs) there is a table showing the RPO and RTO for the different consistency levels. RPO is *recovery point objective* and RTO is *recovery time objective*. Basically, RTO is the time it takes to recover from a failed state to fully operational. RPO is the amount of possible data loss due to the failed state. With a single region it is possible to take up to a week to be fully operational and possibly up to 4 hours (240 minutes) of data loss. For some applications this is tolerable. For many other applications this is risk that must be mitigated. For applications that cannot tolerate the downtime and/or possible data loss then using multiple regions is the next best answer.

With the Strong consistency Level, data is either replicated or it is not. When using multiple regions, the RPO is zero, so no data loss. However, with the other consistency levels there is a chance of data loss. With such low latency to replicate data to other regions, how is it possible to have data loss up to 15 minutes (Session, Consistent Prefix, Eventual) or even up to *K & T* for Bounded Staleness? 

Azure Cosmos DB is an Azure Service Fabric application spread across many servers and across regions. Because of the high availability of Service Fabric an outage at this level does not happen. One of the culprits of an outage is network communication glitches between regions. These glitches occur many times a day and network routers and switches resend data packets as needed. When network communication is challenged for a larger amount of time, seconds to minutes, then data cannot be replicated to other regions during this time.

Consider this scenario; an application is using Cosmos DB and both are in the South Central US region. Global replication is enabled to the North Central US region and Bounded Staleness is the default consistency level. Network communication partition has now occurred but for only 45 seconds. The application never notices the brief outage but has continued to Write data to Cosmos DB. After the connection to the North Central US region has been reestablished the replication gets caught up. 

Now for a more drastic scenario. The outage has now lasted many minutes. The *K & T* for the Bounded Staleness consistency level is at the minimum for multi-region; 100,000 versions and 5 minutes. At what point in the outage should the Cosmos DB instance be told to no longer accept Writes? Until this point the Cosmos DB system believes it come back online and replicate. However, in severe outages, Cosmos DB instances are told to stop accepting Write operations when the outage duration has exceeded the *T* or the number of operations has exceeded *K*. The time between start of the outage to when Cosmos DB is no longer accepting Writes ***is*** the RPO window. 

Now that data center is back online and fully operational. However, two regions are now out-of-sync with the data. Cosmos DB will replicate data to other regions as expected ***then*** become available for applications to connect. During this time of replication [Conflict Resolution Policies](conflict-resolution-policies) come into action. However, should the Cosmos DB instance become in a state where the data written and not replicated cannot be retrieved, there is data loss.

How can developers mitigate this possible data loss? This is where [Custom Synchronization](how-to-multi-master.md) comes in. An application can Write to one region then, using a Session Token, Read from another region. In order to receive a Session Token the Write operation succeeded. If the Read succeeds then data has been replicated. If the Read fails after a timeout period then the application can react as needed. Perhaps it retries the Read. In cases where the data is not replicated, the application can stop writing data to minimize possible data loss. This allows the application to be more aware of the replication issues. 




